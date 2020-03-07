---
title: 12 Avro
toc: false
date: 2017-10-30
---


Apache Avro is a language-neutral data serialization system. The project was created by Doug Cutting to address the major downside of Hadoop <C>Writables</C>: **lack of language portability** [see [Hadoop IO](5 Hadoop IO.md/#3-serialization)]. Having a data format that can be processed by many languages makes it easier to share datasets with a wider audience.

Avro data is described using a language-independent **schema**(模式). Schemas are usually written in JSON, and data is usually encoded using a binary format.

### Geting start

This section based on [Apache Avro™ Getting Started (Java)](http://avro.apache.org/docs/current/gettingstartedjava.html).

#### Defining a schema

Avro schemas are defined using JSON. Schemas are composed of primitive types (null, boolean, int, long, float, double, bytes, and string) and complex types (record, enum, array, map, union, and fixed).

Let's start with a simple schema example, `user.avsc`:

```
{"namespace": "example.avro",
 "type": "record",
 "name": "User",
 "fields": [
     {"name": "name", "type": "string"},
     {"name": "favorite_number",  "type": ["int", "null"]},
     {"name": "favorite_color", "type": ["string", "null"]}
 ]
}
```

This schema defines a record representing a hypothetical user. (Note that a schema file can only contain a single schema definition.) At minimum, a record definition must include its type, a name, and fields. Fields are defined via an array of objects, each of which defines a name and type; The type attribute of a field is another schema object, which can be either a primitive or complex type.


#### Serializing and deserializing

Let's  create some users, serialize them to a data file on disk, and then read back the file and deserialize the users objects.

<hh>Creating users</hh>

First, we use a <C>Parser</C> to read our schema definition and create a <C>Schema</C> object.

```Java
Schema schema = new Schema.Parser().parse(new File("user.avsc"));
```

Using this schema, let's create some users.

```Java
GenericRecord user1 = new GenericData.Record(schema);
user1.put("name", "Alyssa");
user1.put("favorite_number", 256);
// Leave favorite color null

GenericRecord user2 = new GenericData.Record(schema);
user2.put("name", "Ben");
user2.put("favorite_number", 7);
user2.put("favorite_color", "red");
```

We use <C>GenericRecords</C> to represent users. <C>GenericRecords</C>  uses the schema to verify that we only specify valid fields. If we try to set a non-existent field (e.g., user1.put("favorite_animal", "cat")), we'll get an <C>AvroRuntimeException</C> when we run the program.

Note that we do not set user1's favorite color. Since that record is of type ["string", "null"], we can either set it to a string or leave it null; it is essentially optional.

<hh>Serializing</hh>

Now that we've created our user objects, we use generic readers and writers to serialize and deserialize them.

First we'll serialize our users to a data file on disk.

```Java
// Serialize user1 and user2 to disk
File file = new File("users.avro");
DatumWriter<GenericRecord> datumWriter = 
    new GenericDatumWriter<GenericRecord>(schema);
DataFileWriter<GenericRecord> dataFileWriter = 
    new DataFileWriter<GenericRecord>(datumWriter);
dataFileWriter.create(schema, file);
dataFileWriter.append(user1);
dataFileWriter.append(user2);
dataFileWriter.close();
```
 
        
We create a <C>DatumWriter</C>, which converts Java objects into an in-memory serialized format. <C>GenericDatumWriter</C> requires the schema both to determine how to write the <C>GenericRecords</C> and to verify that all non-nullable fields are present.

We also create a <C>DataFileWriter</C>, which writes the serialized records, as well as the schema, to the file specified in the <C>dataFileWriter.create</C> call. We write our users to the file via calls to the <C>dataFileWriter.append</C> method. When we are done writing, we close the data file.

<hh>Deserializing</hh>

Finally, we'll deserialize the data file we just created.

```Java
// Deserialize users from disk
DatumReader<GenericRecord> datumReader = new GenericDatumReader<GenericRecord>(schema);
DataFileReader<GenericRecord> dataFileReader = new DataFileReader<GenericRecord>(file, datumReader);
GenericRecord user = null;
while (dataFileReader.hasNext()) {
    // Reuse user object by passing it to next(). This saves us from
    // allocating and garbage collecting many objects for files with
    // many items.
    user = dataFileReader.next(user);
    System.out.println(user);
}
```

This outputs:

```
{"name": "Alyssa", "favorite_number": 256, "favorite_color": null}
{"name": "Ben", "favorite_number": 7, "favorite_color": "red"}
```
    
Deserializing is very similar to serializing. We create a <C>GenericDatumReader</C>, analogous to the <C>GenericDatumWriter</C> we used in serialization, which converts in-memory serialized items into <C>GenericRecords</C>. We pass the <C>DatumReader</C> and the previously created File to a <C>DataFileReader</C>, analogous to the <C>DataFileWriter</C>, which reads the data file on disk.

Next, we use the <C>DataFileReader</C> to iterate through the serialized users and print the deserialized object to stdout. Note how we perform the iteration: we create a single <C>GenericRecord</C> object which we store the current deserialized user in, and pass this record object to every call of <C>dataFileReader.next</C>. This is a performance optimization that allows the <C>DataFileReader</C> to reuse the same record object rather than allocating a new <C>GenericRecord</C> for every iteration, which can be very expensive in terms of object allocation and garbage collection if we deserialize a large data file. While this technique is the standard way to iterate through a data file, it's also possible to use `:::Java for (GenericRecord user : dataFileReader)` if performance is not a concern.

### Sort Order

Avro defines a sort order for objects. All types except <C>record</C> have preordained rules for their sort order, as described in the Avro specification, that cannot be overridden by the user. For records, however, you can control the sort order by specifying the order attribute for a field. It takes one of three values: <C>ascending</C>(the default), <C>descending</C>(to reverse the order), or <C>ignore</C>(so the field is skipped for comparison purposes)

For example, the following schema (`SortedStringPair.avsc`) defines an ordering of <C>StringPair</C> records by the right field in descending order. The left field is ignored for the purposes of ordering, but it is still present in the projection:

```
{
"type": "record", 
"name": "StringPair",
"doc": "A pair of strings, sorted by right field descending.", 
"fields": [
    {"name": "left", "type": "string", "order": "ignore"},
    {"name": "right", "type": "string", "order": "descending"} ]
}
```

Avro implements efficient binary comparisons. That is to say, Avro does not have to deserialize binary data into objects to perform the comparison, because it can instead work directly on the byte streams. Avro provides the comparator for us.

### Avro MapReduce

Avro provides a number of classes for making it easy to run MapReduce programs on Avro data. 

Let’s rework the MapReduce program for finding the maximum temperature for each year in the weather dataset, this time using the Avro MapReduce API. We will represent weather records using the following schema:

```
{
    "type": "record",
    "name": "WeatherRecord", 
    "doc": "A weather reading.", 
    "fields": [
        {"name": "year", "type": "int"},
        {"name": "temperature", "type": "int"},
        {"name": "stationId", "type": "string"} ]
}
```




There are a couple of differences from the regular Hadoop MapReduce API. The first is the use of wrappers around Avro Java types. The second major difference from regular MapReduce is the use of <C>AvroJob</C> for configuring the <C>job</C>. <C>AvroJob</C> is a convenience class for specifying the Avro schemas for the input, map output, and final output data.

<details>
<summary>Avro Mapreduce
</summary>
```Java
//vv AvroGenericMaxTemperature
public class AvroGenericMaxTemperature extends Configured implements Tool {
  
  private static final Schema SCHEMA = new Schema.Parser().parse(
      "{" +
      "  \"type\": \"record\"," +
      "  \"name\": \"WeatherRecord\"," +
      "  \"doc\": \"A weather reading.\"," +
      "  \"fields\": [" +
      "    {\"name\": \"year\", \"type\": \"int\"}," +
      "    {\"name\": \"temperature\", \"type\": \"int\"}," +
      "    {\"name\": \"stationId\", \"type\": \"string\"}" +
      "  ]" +
      "}"
  );

  public static class MaxTemperatureMapper
      extends Mapper<LongWritable, Text, AvroKey<Integer>,
            AvroValue<GenericRecord>> {
    private NcdcRecordParser parser = new NcdcRecordParser();
    private GenericRecord record = new GenericData.Record(SCHEMA);

    @Override
    protected void map(LongWritable key, Text value, Context context)
        throws IOException, InterruptedException {
      parser.parse(value.toString());
      if (parser.isValidTemperature()) {
        record.put("year", parser.getYearInt());
        record.put("temperature", parser.getAirTemperature());
        record.put("stationId", parser.getStationId());
        context.write(new AvroKey<Integer>(parser.getYearInt()),
            new AvroValue<GenericRecord>(record));
      }
    }
  }
  
  public static class MaxTemperatureReducer
      extends Reducer<AvroKey<Integer>, AvroValue<GenericRecord>,
            AvroKey<GenericRecord>, NullWritable> {

    @Override
    protected void reduce(AvroKey<Integer> key, Iterable<AvroValue<GenericRecord>>
        values, Context context) throws IOException, InterruptedException {
      GenericRecord max = null;
      for (AvroValue<GenericRecord> value : values) {
        GenericRecord record = value.datum();
        if (max == null || 
            (Integer) record.get("temperature") > (Integer) max.get("temperature")) {
          max = newWeatherRecord(record);
        }
      }
      context.write(new AvroKey(max), NullWritable.get());
    }

    private GenericRecord newWeatherRecord(GenericRecord value) {
      GenericRecord record = new GenericData.Record(SCHEMA);
      record.put("year", value.get("year"));
      record.put("temperature", value.get("temperature"));
      record.put("stationId", value.get("stationId"));
      return record;
    }
  }

  @Override
  public int run(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.printf("Usage: %s [generic options] <input> <output>\n",
          getClass().getSimpleName());
      ToolRunner.printGenericCommandUsage(System.err);
      return -1;
    }

    Job job = new Job(getConf(), "Max temperature");
    job.setJarByClass(getClass());

    job.getConfiguration().setBoolean(
        Job.MAPREDUCE_JOB_USER_CLASSPATH_FIRST, true);

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    AvroJob.setMapOutputKeySchema(job, Schema.create(Schema.Type.INT));
    AvroJob.setMapOutputValueSchema(job, SCHEMA);
    AvroJob.setOutputKeySchema(job, SCHEMA);

    job.setInputFormatClass(TextInputFormat.class);
    job.setOutputFormatClass(AvroKeyOutputFormat.class);

    job.setMapperClass(MaxTemperatureMapper.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    return job.waitForCompletion(true) ? 0 : 1;
  }
  
  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new AvroGenericMaxTemperature(), args);
    System.exit(exitCode);
  }
}
```
</details>

#### Sorting Using Avro MapReduce
To sort an Avro datafile, it is simple. The mapper simply emits the input key wrapped in an <C>AvroKey</C> and an <C>AvroValue</C>. The reducer acts as an identity, passing the values through as output keys, which will get written to an Avro datafile.

The sorting happens in the MapReduce shuffle, and the sort function is determined by the Avro schema that is passed to the program. See Chapter 7, section [Shuffle and Sort](ch7/#shuffle-and-sort) for details.

### Useful resources

* [Apache Avro project page](http://avro.apache.org/)
* [CDH usage page for Avro](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/5.0/CDH5-Installation-Guide/cdh5ig_avro_usage.html)
* [Avro Specification](http://avro.apache.org/docs/current/spec.html)


