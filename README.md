<img width="1175" height="431" alt="image" src="https://github.com/user-attachments/assets/53e906b2-d9f2-4633-8392-d48145593fb4" />


# OneConnect

**OneConnect** is a real-time data integration platform that connects to SAP
systems and streams data to multiple destinations with minimal latency. It
enables organizations to unlock the value of their SAP data by making it
available in real time to downstream systems — such as data lakes, analytics
platforms, and third-party applications without relying on traditional
batch extraction methods.

The platform is built around two core components:

- **SAP Data Modeler**: models, designs, and structures SAP data for
  extraction, defining which entities, tables, and views are exposed for
  integration.
- **Smart Gateway**: acts as the connectivity bridge, capturing changes
  from SAP in real time and routing them to one or multiple target
  destinations.

Together, these components replace slow, resource-intensive batch jobs with
a continuous, event-driven flow of data, supporting use cases like
real-time analytics, operational reporting, data replication to cloud
platforms, and synchronization across enterprise systems, all while
reducing the load on SAP's core infrastructure.

This repository serves as the central documentation hub for the OneConnect
suite, bringing together installation, configuration, operation, and
troubleshooting guides in a single place, organized by product.

## Contents

The repository is organized into two main folders, one per suite component:

### [SAP Data Modeler](./SAP_Data_Modeler)

Tool for modeling and designing data structures oriented to SAP environments.
This folder includes:

- Installation guide
- Initial configuration manual
- Use cases and best practices
- Troubleshooting for common issues

### [Smart Gateway](./Smart_Gateway)

Connectivity and integration component that acts as a bridge between SAP
systems and target destinations, enabling real-time data flow. This folder
includes:

- Installation guide and prerequisites
- Connection configuration
- Administration manual
- Troubleshooting for common issues

## How to use this repository

1. Navigate to the folder of the product you're interested in
   (**Data Modeler** or **Smart Gateway**).
2. Start with the `README.md` inside the folder, which serves as the index.
3. Open the specific manual you need to consult.

All documents are in Markdown (`.md`) format, so they render directly on
GitHub with no need to download them.

_Maintained by [Onibex](https://github.com/onibex)._
