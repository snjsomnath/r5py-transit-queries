# Efficient transit queries from r5py

#### Problem Statement:
The objective was to explore different strategies to create large random transit time queries from a r5py traveltime matrix.
This includes writing a large dataset to an HDF5 file efficiently, querying id's from points, finding a closest point and finally querying transit times form two random points.

## Experiment 1 - Storing travel time matrix
The dataset, initially stored in a CSV file, contained a significant number of rows, with key columns like `from_id`, `to_id`, and `travel_time`. The challenge was to optimize the HDF5 file creation process considering factors like processing time, file size, and compression ratio.

### Experimental Setup:
Three different approaches were tested:
1. **Direct Writing Through Pandas**: The first experiment involved writing the dataset directly to an HDF5 file using Pandas, without any preprocessing like NaN filtering or datatype conversion.
   
2. **Using Chunking and Compression**: The second experiment focused on preprocessing the data by filtering out NaN values, setting appropriate datatypes to save memory, and then writing the HDF5 file with both chunking and compression.

3. **Compression Without Chunking**: The final experiment also involved preprocessing the data (NaN filtering and datatype conversion) but used compression without chunking when writing the HDF5 file.

## Results:

Here's a markdown table summarizing the results of the three experiments:

| Strategy                                  | CSV File Size (bytes) | Total Rows Before Filtering | Total Rows After Filtering | Processing Time (seconds) | Writing Time (seconds) | HDF5 File Size (bytes) | Compression Ratio |
|-------------------------------------------|-----------------------|-----------------------------|----------------------------|--------------------------|-----------------------|------------------------|-------------------|
| Direct Writing Through Pandas             | 1,923,541,554         | 85,285,225                  | N/A                        | 33.42                    | 5.94                  | 43,748,297             | 43.97             |
| Using Chunking and Compression            | 1,923,541,554         | 85,285,225                  | 6,383,010                  | 35.23                    | 1.73                  | 16,350,848             | 117.64            |
| Compression Without Chunking              | 1,923,541,554         | 85,285,225                  | 6,383,010                  | 34.65                    | 0.67                  | 6,701,225              | 287.04            |

This table provides a clear comparison of the different strategies, highlighting the impact of preprocessing, chunking, and compression on the efficiency of writing large datasets to HDF5 files.

## Experiment 2 - Querying

Now we setup a simple object that turns this hdf5 into a dict for fast querying

Obviously the fastest load time comes from the hdf5 with the best compression

### Experiment Overview

In this experiment, we explored the efficiency of querying origin-destination (OD) pairs from a preprocessed OD matrix stored in an optimized data structure. The primary goal was to assess the performance scalability of the query process as the number of queried pairs increased. This is crucial for applications requiring rapid access to OD travel times, such as dynamic routing algorithms, traffic simulation models, and various geospatial analyses.

### Results

The results are summarized in the table below, showing both the total time taken to query a set number of OD pairs and the average time per individual query:

| Number of Queries | Total Query Time (seconds) | Average Time Per Call (seconds) |
|-------------------|----------------------------|---------------------------------|
| 1                 | 0.000005                   | 0.000005                        |
| 10                | 0.000023                   | 0.000002                        |
| 100               | 0.000197                   | 0.000002                        |
| 1,000             | 0.001486                   | 0.000001                        |
| 10,000            | 0.012771                   | 0.000001                        |

## Experiment 3: Querying Closest Points

In this experiment, we tested two methods for querying the closest points to a set of random coordinates using a dataset of origin points. The two methods compared are:

1. **get_closest_id function:** This method calculates the distance between each random coordinate and all origin points in the dataset and returns the ID of the closest origin point.

2. **get_closest_id_tree function:** This method uses a BallTree spatial indexing structure to efficiently find the closest origin point to each random coordinate and returns the ID of the closest origin point.

We measured the total time taken to query a varying number of random points and calculated the average time per query call for each method.

### Results

| Number of Random Points | get_closest_id (seconds) | get_closest_id_tree (seconds) |
|-------------------------|--------------------------|-------------------------------|
| 1                       | 0.015549                 | 0.000355                      |
| 10                      | 0.140009                 | 0.001297                      |
| 100                     | 1.309974                 | 0.013414                      |
| 1000                    | 12.895093                | 0.111098                      |
| 9000                    | 115.895936               | 1.005964                      |

## Experiment 4 -  Combining it all

Finally, what we really need is a way to get transit times between two random points.

| Query Count | Total Time (seconds) | Average Time per Call (seconds) |
|-------------|-----------------------|---------------------------------|
| 1           | 2.058234              | 2.058234                        |
| 10          | 2.055068              | 0.205507                        |
| 100         | 2.097684              | 0.020977                        |
| 1000        | 2.015695              | 0.002016                        |
| 9000        | 2.032059              | 0.000226                        |

This table summarizes the total time taken and the average time per call for querying different numbers of random OD pairs. While the total time remains the same, the average time per call is going down. This indicates that there is a 2 second overhead in the querying function.

But this can be avoided if you don't want the overhead of objectifying this process.


# The final working code:
```python 
class ODMatrix:
    def __init__(self, hdf5_path,origins_gdf):
        self.hdf5_path = hdf5_path
        self.travel_times_dict = {}
        self.origins_gdf = origins_gdf
        self.points = np.array(self.origins_gdf[['geometry']].apply(lambda x: [x[0].x, x[0].y], axis=1).tolist())
        self.tree = KDTree(self.points)
        self.ids = self.origins_gdf['id'].to_numpy()
        self.preprocess_data()


    def preprocess_data(self):
        """Preprocess the data from the HDF5 file and store it in a dictionary for fast lookups."""
        with h5py.File(self.hdf5_path, 'r') as hdf5_file:


            # Proceed with the existing logic if datasets are present
            dataset_size = hdf5_file['from_id'].shape[0]
            batch_size = 10000  # Adjust based on your system's memory capacity

            for i in range(0, dataset_size, batch_size):
                from_ids = hdf5_file['from_id'][i:i+batch_size]
                to_ids = hdf5_file['to_id'][i:i+batch_size]
                travel_times = hdf5_file['travel_time'][i:i+batch_size]
                
                for from_id, to_id, travel_time in zip(from_ids, to_ids, travel_times):
                    self.travel_times_dict[(from_id, to_id)] = travel_time
    
    def get_closest_id_tree(self,lat,lon):
        dist,closest_idx = self.tree.query([[lon,lat]])
        # Get closest id
        closest_id = self.ids[closest_idx][0][0]
        return closest_id
    
    def query_travel_time(self,o,d):
        # Get closest id
        origin_closest_id = self.get_closest_id_tree(o[1], o[0])
        # Get closest id
        destination_closest_id = self.get_closest_id_tree(d[1], d[0])
        # Query the travel time for a given origin and destination ID pair.
        travel_time = self.travel_times_dict.get((origin_closest_id, destination_closest_id), None)
        return travel_time


# Test the class
od_matrix = ODMatrix(path_to_hdf5, origins)
# Testing the query_travel_time function
time_start = time.time()
travel_time = od_matrix.query_travel_time([o_x,o_y], [d_x,d_y])
# Print time for processing
print(travel_time)

```
