---
title: "CsvHelper truncate my files"
tags: 
- dotnet
- c#
- CsvHelper
---

# CsvHelper seems to trucate my files apparently without reasons

## Actual code

The following code intends to write "incrementally" rows to the target file.

But for some reason, it truncates the file in the middle of the row.

``` csharp

public class CsvWriteService : ICsvWriteService, IDisposable
{
    readonly CsvWriter _csv;
    readonly StreamWriter _stringWriter;

    public CsvWriteService(IOptions<ClientConfiguration> configuration)
    {
        var clientConfiguration = configuration.Value;
        var outputFilePath = GetFileName(clientConfiguration.CsvFilePath);
        _stringWriter = new StreamWriter(outputFilePath);
        _csv = new CsvWriter(_stringWriter, CultureInfo.InvariantCulture);
    }

    public void Export(PeopleWriteData csvOutput)
    {
        _csv.WriteRecord(csvOutput);
        _csv.NextRecord();
    }

    static string GetFileName(string inputFilePath)
    {
        var outputFilePath = inputFilePath.Replace(".csv", "_out.csv");
        int count = 0;
        while (File.Exists(outputFilePath))
        {
            count++;
            outputFilePath = inputFilePath.Replace(".csv", $"_out_{count}.csv");
        }
        return outputFilePath;
    }

    public void Dispose()
    {
        _stringWriter.Flush();
    }
}

```

After several attempts, I've figured out that CsvHelper truncates the data due to some objects being disposed before the completion of the file write.

## First Refactoting

To avoid data loss, I've managed to rewrite the code as the following, where the file is Opened in append mode and a new row is written on every iteration.

This works but it is slow...

``` csharp
public class CsvWriteService(IOptions<ClientConfiguration> configuration) : ICsvWriteService
{
    readonly string _outputFilePath = GetFileName(configuration.Value.CsvFilePath);

    readonly CsvConfiguration _config = new(CultureInfo.InvariantCulture) { HasHeaderRecord = false };

    public void Export(PeopleWriteData csvOutput)
    {
        using var stream = File.Open(_outputFilePath, FileMode.Append, FileAccess.Write, FileShare.ReadWrite);
        using var stringWriter = new StreamWriter(stream);
        using var csv = new CsvWriter(stringWriter, _config);

        csv.WriteRecord(csvOutput);
        csv.NextRecord();
    }

    static string GetFileName(string inputFilePath)
    {
        var outputFilePath = inputFilePath.Replace(".csv", "_out.csv");
        int count = 0;
        while (File.Exists(outputFilePath))
        {
            count++;
            outputFilePath = inputFilePath.Replace(".csv", $"_out_{count}.csv");
        }
        return outputFilePath;
    }
}
```

## Final Refactoring (for now)

The best way I've found that write all data at the end and speed it up. It's not the best solution but it makes the job done and doesn't drop any data.

``` csharp
public class CsvWriteService(IOptions<ClientConfiguration> configuration) : ICsvWriteService
{
    readonly string _outputFilePath = GetFileName(configuration.Value.CsvFilePath);

    readonly CsvConfiguration _config = new(CultureInfo.InvariantCulture) { HasHeaderRecord = false };

    public void Export(List<PeopleWriteData> csvOutput)
    {
        using var stream = File.Open(_outputFilePath, FileMode.Append, FileAccess.Write, FileShare.ReadWrite);
        using var stringWriter = new StreamWriter(stream);
        using var csv = new CsvWriter(stringWriter, _config);

        csv.WriteRecords(csvOutput);
    }

    static string GetFileName(string inputFilePath)
    {
        var outputFilePath = inputFilePath.Replace(".csv", "_out.csv");
        int count = 0;
        while (File.Exists(outputFilePath))
        {
            count++;
            outputFilePath = inputFilePath.Replace(".csv", $"_out_{count}.csv");
        }
        return outputFilePath;
    }
}
```
