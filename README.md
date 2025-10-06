# Asset Manager Application

A Spring Boot-based asset management system for uploading, storing, and processing image files with automatic thumbnail generation.

## Architecture

The application consists of two main components:

- **Web Module** (`assets-manager-web`): Frontend and API for file upload/management
- **Worker Module** (`assets-manager-worker`): Background service for thumbnail generation

### Technology Stack

- **Backend:** Spring Boot 3.4.3, Java 11
- **Storage:** AWS S3 (configurable with local file storage for development)
- **Message Queue:** RabbitMQ
- **Database:** PostgreSQL
- **Build Tool:** Maven

## Getting Started

### Prerequisites

- Java 11 or higher
- Maven 3.6+
- Docker (for PostgreSQL and RabbitMQ)

### Running Locally

#### Linux/Mac

```bash
cd scripts
./start.sh
```

#### Windows

```cmd
cd scripts
start.cmd
```

This will:
1. Start PostgreSQL container
2. Start RabbitMQ container
3. Launch the web module on port 8080
4. Launch the worker module on port 8081

### Access Points

- **Web Application:** http://localhost:8080
- **Worker Health:** http://localhost:8081/actuator/health
- **RabbitMQ Management:** http://localhost:15672 (guest/guest)

### Configuration

Configuration files are located in:
- `web/src/main/resources/application.properties`
- `worker/src/main/resources/application.properties`

#### Development Profile (Local File Storage)

By default, the application runs with the `dev` profile, which uses local file storage instead of AWS S3.

#### Production Profile (AWS S3)

To use AWS S3, update the properties file with your AWS credentials:

```properties
aws.accessKey=your-access-key
aws.secretKey=your-secret-key
aws.region=us-east-1
aws.s3.bucket=your-bucket-name
```

## Building the Application

```bash
# Build all modules
./mvnw clean package

# Build specific module
cd web
../mvnw clean package
```

## Features

- Upload images to cloud storage (S3 or local filesystem)
- Automatic thumbnail generation via background worker
- View and manage uploaded assets
- Metadata storage in PostgreSQL
- Asynchronous processing with RabbitMQ

## Project Structure

```
asset-manager-ma/
├── web/                    # Web application module
│   └── src/
│       └── main/
│           ├── java/       # Java source code
│           └── resources/  # Configuration and templates
├── worker/                 # Background worker module
│   └── src/
│       └── main/
│           ├── java/       # Java source code
│           └── resources/  # Configuration
├── scripts/                # Startup scripts
│   ├── start.sh           # Linux/Mac startup
│   ├── start.cmd          # Windows startup
│   ├── stop.sh            # Linux/Mac shutdown
│   └── stop.cmd           # Windows shutdown
├── pom.xml                # Parent POM
└── README.md              # This file
```

## Azure Migration

📘 **Planning to migrate to Azure?**

We've prepared comprehensive documentation to help you migrate this application from AWS to Azure:

- **[Azure Migration Assessment Report](AZURE_MIGRATION_ASSESSMENT.md)** - Complete migration analysis including:
  - Service mapping (AWS → Azure)
  - Detailed code changes
  - Cost analysis
  - Deployment architecture
  - 6-week migration plan
  - Risk assessment and rollback procedures

- **[Azure Migration Quick Reference](AZURE_MIGRATION_QUICK_REFERENCE.md)** - Developer quick reference with:
  - Side-by-side code examples (AWS vs Azure)
  - Configuration changes
  - Deployment commands
  - Troubleshooting guide

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

See [LICENSE](LICENSE) file for details.

## Support

For issues or questions, please open an issue in the GitHub repository.
