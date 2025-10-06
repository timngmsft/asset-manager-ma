# Azure Migration Quick Reference Guide
## Asset Manager Application

This is a quick reference guide for developers working on the Azure migration. For the full assessment report, see [AZURE_MIGRATION_ASSESSMENT.md](AZURE_MIGRATION_ASSESSMENT.md).

---

## Quick Service Mapping

| Current (AWS) | Future (Azure) | Changes Required |
|---------------|----------------|------------------|
| AWS S3 | Azure Blob Storage | SDK change, API changes |
| RabbitMQ | Azure Service Bus | SDK change, or keep RabbitMQ |
| PostgreSQL | Azure Database for PostgreSQL | Connection string only |
| EC2/ECS | Azure App Service / Container Apps | Deployment configuration |

---

## Dependency Changes

### Maven Dependencies to Remove

```xml
<!-- Remove -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
</dependency>
```

### Maven Dependencies to Add

```xml
<!-- Add -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-storage-blob</artifactId>
    <version>12.25.0</version>
</dependency>

<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-messaging-servicebus</artifactId>
    <version>7.15.0</version>
</dependency>

<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity</artifactId>
    <version>1.11.0</version>
</dependency>

<!-- Optional: Application Insights -->
<dependency>
    <groupId>com.microsoft.azure</groupId>
    <artifactId>applicationinsights-spring-boot-starter</artifactId>
    <version>3.4.19</version>
</dependency>
```

---

## Configuration Properties Changes

### application.properties

**Before (AWS):**
```properties
# AWS S3
aws.accessKey=your-access-key
aws.secretKey=your-secret-key
aws.region=us-east-1
aws.s3.bucket=your-bucket-name

# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/assets_manager
```

**After (Azure):**
```properties
# Azure Blob Storage
azure.storage.account-name=yourstorageaccount
azure.storage.connection-string=DefaultEndpointsProtocol=https;AccountName=yourstorageaccount;AccountKey=...
azure.storage.container-name=assets-container

# OR use endpoint + managed identity (recommended for production)
azure.storage.blob-endpoint=https://yourstorageaccount.blob.core.windows.net

# Azure Service Bus
azure.servicebus.connection-string=Endpoint=sb://yournamespace.servicebus.windows.net/...
azure.servicebus.queue-name=image-processing-queue

# Azure Database for PostgreSQL
spring.datasource.url=jdbc:postgresql://yourserver.postgres.database.azure.com:5432/assets_manager
spring.datasource.username=adminuser@yourserver
spring.datasource.password=yourpassword

# Optional: Application Insights
azure.application-insights.instrumentation-key=${APPINSIGHTS_INSTRUMENTATIONKEY}
```

---

## Code Changes Cheat Sheet

### 1. Client Initialization

**AWS S3:**
```java
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.auth.credentials.*;
import software.amazon.awssdk.regions.Region;

@Bean
public S3Client s3Client() {
    return S3Client.builder()
            .region(Region.of(region))
            .credentialsProvider(StaticCredentialsProvider.create(
                AwsBasicCredentials.create(accessKey, secretKey)))
            .build();
}
```

**Azure Blob:**
```java
import com.azure.storage.blob.*;
import com.azure.identity.DefaultAzureCredentialBuilder;

@Bean
public BlobServiceClient blobServiceClient() {
    // Option 1: Connection string
    return new BlobServiceClientBuilder()
            .connectionString(connectionString)
            .buildClient();
    
    // Option 2: Managed Identity (recommended)
    return new BlobServiceClientBuilder()
            .endpoint(blobEndpoint)
            .credential(new DefaultAzureCredentialBuilder().build())
            .buildClient();
}

@Bean
public BlobContainerClient blobContainerClient(BlobServiceClient client) {
    return client.getBlobContainerClient(containerName);
}
```

### 2. Upload File

**AWS S3:**
```java
PutObjectRequest request = PutObjectRequest.builder()
        .bucket(bucketName)
        .key(key)
        .contentType(contentType)
        .build();
s3Client.putObject(request, RequestBody.fromInputStream(inputStream, size));
```

**Azure Blob:**
```java
BlobClient blobClient = blobContainerClient.getBlobClient(blobName);
blobClient.upload(inputStream, size, true);

// Set metadata
Map<String, String> metadata = new HashMap<>();
metadata.put("contentType", contentType);
blobClient.setMetadata(metadata);
```

### 3. Download File

**AWS S3:**
```java
GetObjectRequest request = GetObjectRequest.builder()
        .bucket(bucketName)
        .key(key)
        .build();
InputStream stream = s3Client.getObject(request);
```

**Azure Blob:**
```java
BlobClient blobClient = blobContainerClient.getBlobClient(blobName);
InputStream stream = blobClient.openInputStream();
```

### 4. Delete File

**AWS S3:**
```java
DeleteObjectRequest request = DeleteObjectRequest.builder()
        .bucket(bucketName)
        .key(key)
        .build();
s3Client.deleteObject(request);
```

**Azure Blob:**
```java
BlobClient blobClient = blobContainerClient.getBlobClient(blobName);
blobClient.delete();
```

### 5. List Files

**AWS S3:**
```java
ListObjectsV2Request request = ListObjectsV2Request.builder()
        .bucket(bucketName)
        .build();
ListObjectsV2Response response = s3Client.listObjectsV2(request);
List<S3Object> objects = response.contents();
```

**Azure Blob:**
```java
PagedIterable<BlobItem> blobs = blobContainerClient.listBlobs();
for (BlobItem blob : blobs) {
    String name = blob.getName();
    long size = blob.getProperties().getContentLength();
}
```

### 6. Generate URL

**AWS S3:**
```java
GetUrlRequest request = GetUrlRequest.builder()
        .bucket(bucketName)
        .key(key)
        .build();
String url = s3Client.utilities().getUrl(request).toString();
```

**Azure Blob (Public URL):**
```java
BlobClient blobClient = blobContainerClient.getBlobClient(blobName);
String url = blobClient.getBlobUrl();
```

**Azure Blob (SAS Token URL):**
```java
import com.azure.storage.blob.sas.*;
import java.time.OffsetDateTime;

BlobClient blobClient = blobContainerClient.getBlobClient(blobName);
BlobSasPermission sasPermission = new BlobSasPermission().setReadPermission(true);
OffsetDateTime expiryTime = OffsetDateTime.now().plusHours(1);
BlobServiceSasSignatureValues sasValues = new BlobServiceSasSignatureValues(expiryTime, sasPermission);
String sasToken = blobClient.generateSas(sasValues);
String url = blobClient.getBlobUrl() + "?" + sasToken;
```

---

## Message Queue Changes

### RabbitMQ → Azure Service Bus

**RabbitMQ (Current):**
```java
// Configuration
@Bean
public Queue queue() {
    return new Queue(QUEUE_NAME, true);
}

// Send
rabbitTemplate.convertAndSend(QUEUE_NAME, message);

// Receive
@RabbitListener(queues = QUEUE_NAME)
public void processMessage(ImageProcessingMessage message) {
    // Process
}
```

**Azure Service Bus:**
```java
// Configuration
@Bean
public ServiceBusSenderClient serviceBusSenderClient(
        @Value("${azure.servicebus.connection-string}") String connectionString,
        @Value("${azure.servicebus.queue-name}") String queueName) {
    return new ServiceBusClientBuilder()
            .connectionString(connectionString)
            .sender()
            .queueName(queueName)
            .buildClient();
}

@Bean
public ServiceBusProcessorClient serviceBusProcessorClient(
        @Value("${azure.servicebus.connection-string}") String connectionString,
        @Value("${azure.servicebus.queue-name}") String queueName) {
    return new ServiceBusClientBuilder()
            .connectionString(connectionString)
            .processor()
            .queueName(queueName)
            .processMessage(this::processMessage)
            .processError(this::processError)
            .buildProcessorClient();
}

// Send
public void sendMessage(ImageProcessingMessage message) {
    String messageBody = objectMapper.writeValueAsString(message);
    serviceBusSenderClient.sendMessage(new ServiceBusMessage(messageBody));
}

// Receive
private void processMessage(ServiceBusReceivedMessageContext context) {
    ServiceBusReceivedMessage message = context.getMessage();
    String body = message.getBody().toString();
    ImageProcessingMessage imageMessage = objectMapper.readValue(body, ImageProcessingMessage.class);
    
    try {
        // Process the message
        // ...
        
        // Complete the message (remove from queue)
        context.complete();
    } catch (Exception e) {
        // Abandon the message (retry later)
        context.abandon();
    }
}

private void processError(ServiceBusErrorContext errorContext) {
    logger.error("Error processing message", errorContext.getException());
}

// Start processor
@PostConstruct
public void startProcessing() {
    serviceBusProcessorClient.start();
}
```

---

## Files to Modify

### Web Module

1. **`pom.xml`** - Update dependencies
2. **`application.properties`** - Update configuration
3. **`AwsS3Config.java`** → Rename to **`AzureBlobConfig.java`**
4. **`AwsS3Service.java`** → Rename to **`AzureBlobStorageService.java`**
5. **`RabbitConfig.java`** - Update for Service Bus (or keep for RabbitMQ)

### Worker Module

1. **`pom.xml`** - Update dependencies
2. **`application.properties`** - Update configuration
3. **`AwsS3Config.java`** → Rename to **`AzureBlobConfig.java`**
4. **`S3FileProcessingService.java`** → Rename to **`AzureBlobFileProcessingService.java`**
5. **`RabbitConfig.java`** - Update for Service Bus (or keep for RabbitMQ)

---

## Testing Checklist

### Local Testing (Dev Profile)

No changes needed - uses `LocalFileStorageService` which remains unchanged.

### Azure Testing

- [ ] Configure Azure resources (Storage Account, Service Bus, PostgreSQL)
- [ ] Update `application.properties` with Azure credentials
- [ ] Test file upload
- [ ] Test file download
- [ ] Test file deletion
- [ ] Test listing files
- [ ] Test thumbnail generation (full workflow)
- [ ] Test error scenarios (network failure, authentication failure)

---

## Deployment Commands

### Azure CLI Setup

```bash
# Login
az login

# Set subscription
az account set --subscription "Your Subscription Name"

# Create resource group
az group create --name asset-manager-rg --location eastus
```

### Deploy Storage Account

```bash
# Create storage account
az storage account create \
  --name assetmgrstore \
  --resource-group asset-manager-rg \
  --location eastus \
  --sku Standard_LRS

# Get connection string
az storage account show-connection-string \
  --name assetmgrstore \
  --resource-group asset-manager-rg

# Create container
az storage container create \
  --name assets-container \
  --account-name assetmgrstore
```

### Deploy Service Bus

```bash
# Create namespace
az servicebus namespace create \
  --name assetmgr-bus \
  --resource-group asset-manager-rg \
  --location eastus \
  --sku Standard

# Create queue
az servicebus queue create \
  --name image-processing-queue \
  --namespace-name assetmgr-bus \
  --resource-group asset-manager-rg

# Get connection string
az servicebus namespace authorization-rule keys list \
  --resource-group asset-manager-rg \
  --namespace-name assetmgr-bus \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv
```

### Deploy PostgreSQL

```bash
# Create server
az postgres flexible-server create \
  --name assetmgr-db \
  --resource-group asset-manager-rg \
  --location eastus \
  --admin-user adminuser \
  --admin-password "YourSecurePassword123!" \
  --sku-name Standard_B1ms

# Create database
az postgres flexible-server db create \
  --resource-group asset-manager-rg \
  --server-name assetmgr-db \
  --database-name assets_manager

# Allow Azure services
az postgres flexible-server firewall-rule create \
  --resource-group asset-manager-rg \
  --name assetmgr-db \
  --rule-name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

### Deploy App Service

```bash
# Create App Service plan
az appservice plan create \
  --name assetmgr-plan \
  --resource-group asset-manager-rg \
  --sku B1 \
  --is-linux

# Create web app
az webapp create \
  --name assetmgr-web \
  --resource-group asset-manager-rg \
  --plan assetmgr-plan \
  --runtime "JAVA:11-java11"

# Deploy (from JAR)
az webapp deploy \
  --resource-group asset-manager-rg \
  --name assetmgr-web \
  --src-path web/target/assets-manager-web-0.0.1-SNAPSHOT.jar \
  --type jar
```

---

## Common Issues & Solutions

### Issue: Connection timeout to PostgreSQL

**Solution:**
```bash
# Add firewall rule for your IP
az postgres flexible-server firewall-rule create \
  --resource-group asset-manager-rg \
  --name assetmgr-db \
  --rule-name AllowMyIP \
  --start-ip-address YOUR_IP \
  --end-ip-address YOUR_IP
```

### Issue: Blob not found errors

**Solution:**
- Verify container exists
- Check blob name format (no leading slash)
- Verify connection string or managed identity permissions

### Issue: Service Bus authentication failure

**Solution:**
- Verify connection string is correct
- Check namespace and queue names
- Ensure namespace is in Standard or Premium tier (not Basic for advanced features)

### Issue: App Service deployment fails

**Solution:**
- Verify JAR is built successfully: `mvn clean package`
- Check Java version matches (Java 11)
- Review deployment logs: `az webapp log tail --name assetmgr-web --resource-group asset-manager-rg`

---

## Performance Tips

1. **Use Managed Identity** instead of connection strings (better security, no rotation needed)
2. **Enable blob caching** for frequently accessed files
3. **Use batch operations** when uploading multiple files
4. **Set appropriate SAS token expiry** (not too long for security, not too short for usability)
5. **Use Azure CDN** for global content delivery (if needed)
6. **Monitor with Application Insights** to identify bottlenecks

---

## Useful Azure CLI Commands

```bash
# View all resources in resource group
az resource list --resource-group asset-manager-rg --output table

# View storage account details
az storage account show --name assetmgrstore --resource-group asset-manager-rg

# List blobs in container
az storage blob list --account-name assetmgrstore --container-name assets-container --output table

# View Service Bus queue details
az servicebus queue show --name image-processing-queue --namespace-name assetmgr-bus --resource-group asset-manager-rg

# View PostgreSQL databases
az postgres flexible-server db list --resource-group asset-manager-rg --server-name assetmgr-db --output table

# View App Service logs
az webapp log tail --name assetmgr-web --resource-group asset-manager-rg

# Clean up all resources
az group delete --name asset-manager-rg --yes --no-wait
```

---

## Environment Variables for Local Testing

```bash
export AZURE_STORAGE_CONNECTION_STRING="DefaultEndpointsProtocol=https;AccountName=..."
export AZURE_STORAGE_CONTAINER_NAME="assets-container"
export AZURE_SERVICEBUS_CONNECTION_STRING="Endpoint=sb://..."
export AZURE_SERVICEBUS_QUEUE_NAME="image-processing-queue"
export SPRING_DATASOURCE_URL="jdbc:postgresql://assetmgr-db.postgres.database.azure.com:5432/assets_manager"
export SPRING_DATASOURCE_USERNAME="adminuser@assetmgr-db"
export SPRING_DATASOURCE_PASSWORD="YourPassword"
```

---

## Next Steps

1. **Read the full assessment:** [AZURE_MIGRATION_ASSESSMENT.md](AZURE_MIGRATION_ASSESSMENT.md)
2. **Set up Azure trial subscription**
3. **Create dev environment** in Azure
4. **Make code changes** as outlined above
5. **Test thoroughly** before production migration
6. **Review security** considerations (use Managed Identity)
7. **Plan data migration** strategy

---

## Support Resources

- [Azure Blob Storage Java SDK Documentation](https://learn.microsoft.com/en-us/java/api/overview/azure/storage-blob-readme)
- [Azure Service Bus Java SDK Documentation](https://learn.microsoft.com/en-us/java/api/overview/azure/messaging-servicebus-readme)
- [Azure Database for PostgreSQL Documentation](https://learn.microsoft.com/en-us/azure/postgresql/)
- [Azure CLI Reference](https://learn.microsoft.com/en-us/cli/azure/)
- [Migration from AWS to Azure](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/)

---

*For questions or issues during migration, refer to the main assessment document or consult Azure documentation.*
