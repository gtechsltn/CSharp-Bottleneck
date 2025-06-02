# Bottleneck in C#

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
