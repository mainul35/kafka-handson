### Common Kafka Questions
*What is Apache Kafka?*
A distributed, fault tolerant and highly scalable message broker and stream processing platform.

*Why Zookeeper is used with Kafka Cluster?*
Zookeper works to orchastrate the kafka Cluster.

*What is Producer?*
A Producer is a client that sends messages to a Kafka topic.

*What is Consumer?*
A Consumer is a client that reads messages from a Kafka topic.

*What is Kafka Broker?*
A Kafka Broker is a server that stores and serves Kafka topics and their messages.

*How does Kafka ensure durability?*
Kafka ensures durability through data replication and persistent storage. Each message is written to disk and replicated across multiple brokers in the cluster. This replication ensures that even if one broker fails, the data is still available on other brokers. Additionally, Kafka uses a commit log to store messages, which allows for recovery in case of failures.

*What is a topic?*
A topic in Kafka is a logical channel to which producers send messages and from which consumers read messages. It acts as a queue where messages are stored and categorized under a specific name, allowing multiple producers and consumers to interact with the data independently.

*What is a partition? How is it related to a Kafka topic?*
A partition is a division of a Kafka topic. Each topic is split into multiple partitions to allow for parallel processing of messages. Partitions enable Kafka to scale horizontally by distributing the data across multiple brokers, and they ensure fault tolerance by replicating data across different brokers.

*Can you explain partition with a real-world application example, like a ride-sharing application?*
In a ride-sharing application, a topic could represent all ride requests. Each partition within this topic could represent ride requests for a specific geographic region. This way, ride requests from different regions can be processed in parallel by different consumers, ensuring that the system can handle a large number of ride requests efficiently. Producers (users requesting rides) send their ride requests to the appropriate partition based on their location, and consumers (drivers) read from the partition corresponding to their region to find nearby ride requests.

*How can partitions help if there is only one location in the ride-sharing app?*
Even if there is only one location, partitions can still help by distributing the load across multiple partitions. For example, ride requests can be distributed across partitions based on the time of the request or the user ID. This allows multiple consumers to process ride requests in parallel, improving the overall throughput and performance of the system. It also provides fault tolerance, as the data is replicated across multiple brokers, ensuring that ride requests are not lost in case of a broker failure.

*Can you explain with a codebase example?*
```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import java.util.UUID;
import java.util.Timer;
import java.util.TimerTask;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class RideSharingService {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper = new ObjectMapper();

    public RideSharingService(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // Producer example: Send ride request
    public void sendRideRequest(String userId, String location, String timestamp) {
        String topic = "ride_requests";
        String requestId = UUID.randomUUID().toString(); // Auto-generated unique request ID
        int partition = requestId.hashCode() % 3; // Assuming 3 partitions
        String rideRequest = String.format("{\"request_id\": \"%s\", \"user_id\": \"%s\", \"location\": \"%s\", \"timestamp\": \"%s\"}", requestId, userId, location, timestamp);
        kafkaTemplate.send(topic, partition, requestId, rideRequest);
    }

    // Consumer example: Listen for ride requests
    @KafkaListener(topics = "ride_requests", groupId = "ride_consumers")
    public void listenForRideRequests(String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
        System.out.println("Processing ride request from partition " + partition + ": " + message);
        // Extract request ID from message and send ride confirmation
        String requestId = extractRequestIdFromMessage(message);
        sendRideConfirmation(requestId, "Driver123");
    }

    // Producer example: Send ride confirmation
    public void sendRideConfirmation(String requestId, String driverId) {
        String topic = "ride_confirmations";
        int partition = requestId.hashCode() % 3; // Assuming 3 partitions
        String rideConfirmation = String.format("{\"request_id\": \"%s\", \"driver_id\": \"%s\"}", requestId, driverId);
        kafkaTemplate.send(topic, partition, requestId, rideConfirmation);
    }

    // Consumer example: Listen for ride confirmations
    @KafkaListener(topics = "ride_confirmations", groupId = "ride_consumers")
    public void listenForRideConfirmations(String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
        System.out.println("Processing ride confirmation from partition " + partition + ": " + message);
        // Extract request ID and driver ID from message and start sending driver location updates
        String requestId = extractRequestIdFromMessage(message);
        String driverId = extractDriverIdFromMessage(message);
        startPeriodicLocationUpdates(requestId);
    }

    // Producer example: Periodically send driver location updates using the unique request ID
    public void sendDriverLocationUpdate(String requestId, double latitude, double longitude) {
        String topic = "driver_location_updates";
        int partition = requestId.hashCode() % 3; // Assuming 3 partitions
        String locationUpdate = String.format("{\"request_id\": \"%s\", \"latitude\": \"%f\", \"longitude\": \"%f\"}", requestId, latitude, longitude);
        kafkaTemplate.send(topic, partition, requestId, locationUpdate);
    }

    // Start periodic location updates
    public void startPeriodicLocationUpdates(String requestId) {
        Timer timer = new Timer();
        TimerTask task = new TimerTask() {
            double latitude = 40.7128; // Starting latitude
            double longitude = -74.0060; // Starting longitude

            @Override
            public void run() {
                sendDriverLocationUpdate(requestId, latitude, longitude);
                latitude += 0.0001; // Move north
                longitude += 0.0001; // Move east
            }
        };
        timer.schedule(task, 0, 1000); // Schedule task to run every 1000ms for 1 minute
        new Timer().schedule(new TimerTask() {
            @Override
            public void run() {
                task.cancel(); // Stop the task after 1 minute
            }
        }, 60000);
    }

    // Helper methods to extract request ID and driver ID from messages
    private String extractRequestIdFromMessage(String message) {
        try {
            JsonNode jsonNode = objectMapper.readTree(message);
            return jsonNode.get("request_id").asText();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    private String extractDriverIdFromMessage(String message) {
        try {
            JsonNode jsonNode = objectMapper.readTree(message);
            return jsonNode.get("driver_id").asText();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```
