# ✈️ FlightRadar-Live-Dashboard
FlightRadar-Live-Dashboard is an interactive real-time dashboard that visualizes global flight data. It provides live tracking of airplanes, including details such as flight number, airline, origin, destination, altitude, and speed. The dashboard continuously updates, giving users an up-to-date view of air traffic around the world.

Built using powerful data visualization tools and APIs, the project allows users to:

 🔹 Monitor active flights in real time
 🔹 Filter data by airline, location, or flight status
 🔹 View key flight metrics (altitude, speed, departure/arrival airports) 
 🔹 Display flight paths on an interactive world map

This project demonstrates the integration of live streaming data, API consumption, and dynamic visualization — making it a great tool for aviation enthusiasts, data analysts, and developers interested in real-time analytics.



# code

    from pyspark.sql import SparkSession
    from pyspark.sql.functions import from_json, col
    from pyspark.sql.types import StructType, StringType, LongType
    import psycopg2
    spark = SparkSession.builder.appName("FlightStream").getOrCreate()
    schema = StructType() \
     .add("flight_id", StringType()) \
     .add("origin", StringType()) \
     .add("destination", StringType()) \
     .add("status", StringType()) \
     .add("departure_time", LongType()) \
     .add("arrival_time", LongType())
    df = spark.readStream \
     .format("kafka") \
     .option("kafka.bootstrap.servers", "broker:29092") \
     .option("subscribe", "flights") \
     .option("startingOffsets", "latest") \
     .load()
    flights = df.select(from_json(col("value").cast("string"), schema).alias("data")).select("data.*")
    def foreach_batch(df, epoch_id):
       conn = psycopg2.connect(
          dbname="flights_project", user="admin", password="admin", host="postgres_general")
    cur = conn.cursor()
    for row in df.collect():
        cur.execute("""
            INSERT INTO flights (flight_id, origin, destination, status, departure_time, arrival_time)
            VALUES (%s,%s,%s,%s,%s,%s)
            ON CONFLICT (flight_id) DO UPDATE
            SET status = EXCLUDED.status, arrival_time = EXCLUDED.arrival_time;
        """, (row.flight_id, str(row.origin), str(row.destination), row.status, row.departure_time, row.arrival_time))
           conn.commit()
           cur.close()
           conn.close()

    query = flights.writeStream.foreachBatch(foreach_batch).start()
    query.awaitTermination()
