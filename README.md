# Bottleneck in C#

## 1. CPU-bound Bottleneck (C#)
```
using System;
using System.Diagnostics;
using System.Numerics; // For BigInteger if you need truly massive numbers

public class CpuBottleneck
{
    public static void SimulateCpuBottleneck(long iterations)
    {
        Console.WriteLine($"Simulating CPU bottleneck with {iterations:N0} iterations...");
        Stopwatch sw = Stopwatch.StartNew();

        double result = 0;
        for (long i = 0; i < iterations; i++)
        {
            // Perform a computationally expensive operation
            // Using Math.Sqrt and Math.Pow on doubles is fairly CPU intensive
            result += Math.Sqrt(i * i + 1) + Math.Pow(i, 0.3333333333333333);
        }

        sw.Stop();
        Console.WriteLine($"CPU Bottleneck completed in {sw.Elapsed.TotalSeconds:F4} seconds. Result (to prevent optimization removal): {result}");
    }

    public static void Main(string[] args)
    {
        // Adjust iterations based on your CPU and desired duration
        // 100,000,000 might take several seconds on a modern CPU
        SimulateCpuBottleneck(100_000_000);

        Console.WriteLine("\nPress any key to exit.");
        Console.ReadKey();
    }
}
``` 

## 2. Memory-bound Bottleneck (C#)
```
using System;
using System.Collections.Generic;
using System.Diagnostics;

public class MemoryBottleneck
{
    private static List<byte[]> _largeDataStore = new List<byte[]>();

    public static void SimulateMemoryBottleneck(int totalSizeMB, int chunkSizeKB = 1024)
    {
        Console.WriteLine($"Simulating Memory bottleneck by allocating {totalSizeMB} MB...");
        Stopwatch sw = Stopwatch.StartNew();

        int chunkSize = chunkSizeKB * 1024; // Convert KB to bytes
        long bytesToAllocate = (long)totalSizeMB * 1024 * 1024; // Convert MB to bytes
        long allocatedBytes = 0;

        try
        {
            while (allocatedBytes < bytesToAllocate)
            {
                byte[] chunk = new byte[chunkSize];
                // Optionally fill the chunk to ensure memory is physically touched, not just reserved
                // Array.Fill(chunk, (byte)0);
                _largeDataStore.Add(chunk);
                allocatedBytes += chunkSize;
                Console.WriteLine($"Allocated {allocatedBytes / (1024 * 1024)} MB...");
            }
            Console.WriteLine($"Successfully allocated {allocatedBytes / (1024 * 1024)} MB.");
        }
        catch (OutOfMemoryException ex)
        {
            Console.WriteLine($"OutOfMemoryException: {ex.Message}");
            Console.WriteLine($"Allocated {allocatedBytes / (1024 * 1024)} MB before OOM.");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"An error occurred: {ex.Message}");
        }

        sw.Stop();
        Console.WriteLine($"Memory Bottleneck simulation completed in {sw.Elapsed.TotalSeconds:F4} seconds.");
        // Keep _largeDataStore in scope to prevent GC from freeing memory immediately
        Console.WriteLine($"Current number of allocated chunks: {_largeDataStore.Count}");
    }

    public static void Main(string[] args)
    {
        // Adjust totalSizeMB based on your available RAM.
        // Be careful: setting this too high might crash your system or IDE!
        // Start with a reasonable amount, e.g., 512 MB or 1024 MB.
        SimulateMemoryBottleneck(1024); // Attempt to allocate 1 GB

        Console.WriteLine("\nPress any key to exit.");
        Console.ReadKey();
    }
}
```

## 3. Disk I/O-bound Bottleneck (C#)
```
using System;
using System.Diagnostics;
using System.IO;
using System.Threading.Tasks;

public class DiskIoBottleneck
{
    public static async Task SimulateDiskIoBottleneck(string filename, int fileSizeMB, int bufferSizeKB = 4)
    {
        Console.WriteLine($"Simulating Disk I/O bottleneck on file: {filename}");
        Console.WriteLine($"File size: {fileSizeMB} MB, Buffer size: {bufferSizeKB} KB");

        long fileSize = (long)fileSizeMB * 1024 * 1024;
        int bufferSize = bufferSizeKB * 1024; // Convert KB to bytes
        byte[] buffer = new byte[bufferSize];
        Random rand = new Random();
        rand.NextBytes(buffer); // Fill buffer with random data

        Stopwatch sw = Stopwatch.StartNew();

        // --- Phase 1: Write a large file ---
        Console.WriteLine("Phase 1: Writing file...");
        try
        {
            using (FileStream fs = new FileStream(filename, FileMode.Create, FileAccess.Write, FileShare.None, bufferSize, FileOptions.Asynchronous))
            {
                long bytesWritten = 0;
                while (bytesWritten < fileSize)
                {
                    await fs.WriteAsync(buffer, 0, buffer.Length);
                    bytesWritten += buffer.Length;
                    // Console.WriteLine($"Written {bytesWritten / (1024 * 1024)} MB..."); // Uncomment for verbose progress
                }
            }
            Console.WriteLine($"Finished writing {fileSizeMB} MB to {filename} in {sw.Elapsed.TotalSeconds:F4} seconds.");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error during write: {ex.Message}");
            return;
        }

        sw.Restart(); // Reset stopwatch for read phase

        // --- Phase 2: Read the file repeatedly ---
        Console.WriteLine("Phase 2: Reading file...");
        try
        {
            int readIterations = 3; // Read the file multiple times to stress I/O
            for (int i = 0; i < readIterations; i++)
            {
                long bytesRead = 0;
                using (FileStream fs = new FileStream(filename, FileMode.Open, FileAccess.Read, FileShare.Read, bufferSize, FileOptions.Asynchronous))
                {
                    while (true)
                    {
                        int bytesActuallyRead = await fs.ReadAsync(buffer, 0, buffer.Length);
                        if (bytesActuallyRead == 0)
                            break;
                        bytesRead += bytesActuallyRead;
                    }
                }
                Console.WriteLine($"Finished read iteration {i + 1}/{readIterations}. Total bytes read: {bytesRead / (1024 * 1024)} MB.");
            }
            Console.WriteLine($"Finished reading {readIterations} times in {sw.Elapsed.TotalSeconds:F4} seconds.");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error during read: {ex.Message}");
        }
        finally
        {
            // Clean up the created file
            if (File.Exists(filename))
            {
                File.Delete(filename);
                Console.WriteLine($"Cleaned up: {filename}");
            }
        }
    }

    public static async Task Main(string[] args)
    {
        string tempFile = Path.Combine(Path.GetTempPath(), "disk_bottleneck_test.bin");
        // Adjust file size. 1024 MB (1 GB) is a good starting point for testing.
        await SimulateDiskIoBottleneck(tempFile, 1024);

        Console.WriteLine("\nPress any key to exit.");
        Console.ReadKey();
    }
}
```

## 4. Network I/O-bound Bottleneck (C#)
```
using System;
using System.Diagnostics;
using System.Net.Http;
using System.Threading.Tasks;
using System.Collections.Generic;
using System.Linq;

public class NetworkIoBottleneck
{
    public static async Task SimulateNetworkBottleneck(string url, int numberOfRequests, int maxConcurrentRequests = 10)
    {
        Console.WriteLine($"Simulating Network I/O bottleneck by making {numberOfRequests:N0} requests to {url}");
        Console.WriteLine($"Max concurrent requests: {maxConcurrentRequests}");

        HttpClient httpClient = new HttpClient(); // HttpClient is designed for reuse
        Stopwatch sw = Stopwatch.StartNew();
        int completedRequests = 0;

        try
        {
            var tasks = new List<Task>();
            for (int i = 0; i < numberOfRequests; i++)
            {
                // Create a task for each request
                var requestTask = Task.Run(async () =>
                {
                    try
                    {
                        HttpResponseMessage response = await httpClient.GetAsync(url);
                        response.EnsureSuccessStatusCode(); // Throws an exception if not a success code (2xx)
                        // Optional: Read content to simulate more data transfer
                        // string content = await response.Content.ReadAsStringAsync();
                        Interlocked.Increment(ref completedRequests);
                    }
                    catch (HttpRequestException e)
                    {
                        Console.WriteLine($"Request failed: {e.Message}");
                    }
                });

                tasks.Add(requestTask);

                // Limit concurrency
                if (tasks.Count >= maxConcurrentRequests)
                {
                    // Wait for any task to complete before adding more
                    Task completed = await Task.WhenAny(tasks);
                    tasks.Remove(completed);
                }
            }

            // Wait for all remaining tasks to complete
            await Task.WhenAll(tasks);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"An unhandled error occurred: {ex.Message}");
        }
        finally
        {
            httpClient.Dispose(); // Dispose HttpClient when done
        }

        sw.Stop();
        Console.WriteLine($"\nNetwork Bottleneck simulation completed.");
        Console.WriteLine($"Successfully completed: {completedRequests:N0} requests.");
        Console.WriteLine($"Total time: {sw.Elapsed.TotalSeconds:F4} seconds.");
    }

    public static async Task Main(string[] args)
    {
        // Use a public URL that allows many requests, or set up your own test server.
        // Be respectful: do not spam public services with excessive requests.
        // "http://example.com" is a good, safe choice for simple requests.
        string targetUrl = "http://example.com";
        int requestsToMake = 500; // Number of requests
        int concurrentLimit = 50; // Max parallel requests

        await SimulateNetworkBottleneck(targetUrl, requestsToMake, concurrentLimit);

        Console.WriteLine("\nPress any key to exit.");
        Console.ReadKey();
    }
}
```
