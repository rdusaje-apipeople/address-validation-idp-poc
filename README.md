# Address Validation and KYC Data Extraction via MuleSoft IDP

This repository contains a Mule 4 application that automates the extraction and validation of Know Your Customer (KYC) details from US Driver's Licenses using **MuleSoft Intelligent Document Processing (IDP)**. The application exposes an HTTP endpoint to accept driver's license images, uploads them to the IDP service, polls for the results, and transforms the extracted fields into a structured JSON response.

---

## 📋 Table of Contents
1. [Executive Summary](#-executive-summary)
2. [Technology Stack](#-technology-stack)
3. [Prerequisites](#-prerequisites)
4. [Step-by-Step Local Deployment Guide](#-step-by-step-local-deployment-guide)
   - [1. Clone the Repository](#1-clone-the-repository)
   - [2. IDP Document Action Setup](#2-idp-document-action-setup)
   - [3. Configure Secure Properties](#3-configure-secure-properties)
   - [4. Import IDP Connector dependency](#4-import-idp-connector-dependency)
   - [5. Run the Application Locally](#5-run-the-application-locally)
5. [Testing & Verification](#-testing--verification)
6. [Data Mapping & Transformation](#-data-mapping--transformation)

---

## 🚀 Executive Summary

Automating identity verification is a critical part of modern user onboarding and compliance pipelines. This project showcases how to eliminate manual entry of PII (Personally Identifiable Information) by converting unstructured government-issued ID documents into a structured, normalized JSON payload. Using MuleSoft IDP, we leverage pre-trained AI models capable of handling varying photo qualities, formats, and layouts, ensuring high-fidelity extraction and deterministic confidence scoring.

---

## 🛠️ Technology Stack

- **Mule Runtime**: `4.9.11`
- **Mule Maven Plugin**: `4.6.0`
- **Anypoint Platform IDP**: AI-powered service for OCR and document structure analysis.
- **Anypoint Studio**: IDE used to design and run the integration flow.
- **Key Dependencies**:
  - `mule-http-connector` (`1.11.0`): Exposes the intake endpoint.
  - `mule-plugin-idp-action` (`1.0.0`): Custom connector representing the published IDP Document Action.
  - `mule-scripting-module` (`2.1.1`) & `groovy-jsr223` (`3.0.19`): Implements an inline delay.
  - `mule-secure-configuration-property-module` (`1.3.1`): Encrypts sensitive Anypoint Client credentials.

---

## 📋 Prerequisites

To run this application locally, ensure you have the following installed and configured:
1. **Anypoint Studio 7.x** (or later) or **VS Code with Mule DX extension**.
2. **Java 8 or Java 11 JDK** (configured in your PATH).
3. **Apache Maven 3.6.x** (or later).
4. An **Anypoint Platform Account** with:
   - Access to **Intelligent Document Processing (IDP)**.
   - Permissions to create a **Connected App** for OAuth 2.0 authentication.

---

## 🖥️ Step-by-Step Local Deployment Guide

### 1. Clone the Repository
Clone this repository to your local machine using terminal:
```bash
git clone https://github.com/apipeople/sfdc-omnistudio-account-application.git
cd sfdc-omnistudio-account-application
```
*(Note: If you have a different repository remote URL, replace it above).*

---

### 2. IDP Document Action Setup

Before running the Mule application, the Intelligent Document Processing service must be configured on your Anypoint Platform:
1. Log in to **Anypoint Platform** and navigate to **Intelligent Document Processing (IDP)**.
2. Create a new **Document Action** (e.g., US Driver's License Extraction).
3. Define the extraction schema by adding the following target fields and recommended confidence thresholds:
   - **FirstName**: Extract the cardholder’s given name. Look for labels `2` or `FN`. (Threshold: `75%`)
   - **LastName**: Extract the cardholder’s surname. Look for labels `1`. (Threshold: `75%`)
   - **LicenseNumber**: Extract the ID number. Parses labels `4d`, `DLN`, `ID`, or `LIC NO`. (Threshold: `80%`)
   - **Address**: Extract the full residential address (street, city, state, ZIP) as a single line. (Threshold: `80%`)
4. Publish the Document Action to **Anypoint Exchange**.

---

### 3. Configure Secure Properties

This application encrypts Connected App credentials in configuration files.
1. Create a **Connected App** in Anypoint Platform (`Access Management > Connected Apps`) with client credentials grant type and scopes to execute IDP actions.
2. Encrypt the Connected App **Client ID** and **Client Secret** using the Mule Artifact Decryption tool (Blowfish algorithm in CBC mode is configured by default).
3. Open `src/main/resources/config.yaml` and update it with your encrypted credentials:
   ```yaml
   idp:
     client_id: "![YOUR_ENCRYPTED_CONNECTED_APP_CLIENT_ID]"
     client_secret: "![YOUR_ENCRYPTED_CONNECTED_APP_CLIENT_SECRET]"
   ```
4. Secure properties are resolved dynamically using an encryption key passed at runtime (e.g., `-Dkey=YOUR_ENCRYPTION_KEY`).

---

### 4. Import IDP Connector dependency
After publishing your IDP Document Action to Anypoint Exchange, you must add it as a dependency in the project's `pom.xml`.
1. Copy the dependency XML from Anypoint Exchange. It will look similar to this:
   ```xml
   <dependency>
       <groupId>YOUR_ORG_ID</groupId>
       <artifactId>mule-plugin-idp-action-extract-us-dl</artifactId>
       <version>1.0.0</version>
       <classifier>mule-plugin</classifier>
   </dependency>
   ```
2. Replace the placeholder dependency in `pom.xml` with your published connector.
3. Refresh the Maven dependencies in Anypoint Studio or execute `mvn clean install` to download the connector.

---

### 5. Run the Application Locally

#### Option A: Running from Anypoint Studio
1. Import the project into Anypoint Studio: `File > Import > Packaged mule application (.jar) or Maven project`.
2. Right-click the project root and select **Run As > Mule Application**.
3. In the Run Configurations window, navigate to the **Arguments** tab.
4. Add the decryption key to the **VM Arguments** box:
   ```bash
   -Dkey=YOUR_DECRYPTION_KEY
   ```
5. Click **Run**. The application will start and expose a local listener on port `8081`.

#### Option B: Running from Command Line (Maven)
You can compile and run the application directly from your terminal:
```bash
mvn clean package
# Run the application locally by providing the decryption key
mvn mule:run -Dkey=YOUR_DECRYPTION_KEY
```

---

## 🧪 Testing & Verification

Once the application displays a `DEPLOYED` status, you can test the address extraction endpoint locally.

- **HTTP Method**: `POST`
- **Local Endpoint**: `http://localhost:8081/extract-address-us-dl`
- **Query Parameter**: `DocumentType=US_DL` (Optional, defaults based on setup)
- **Headers**:
  - `Content-Type`: `image/jpeg` or `image/png`
  - `FileName`: `driving_license.jpg` (or the name of your file)
- **Body**: Binary format (select a driver's license image file)

### Testing via Curl
Submit an identity document using `curl` from your terminal:
```bash
curl -X POST "http://localhost:8081/extract-address-us-dl?DocumentType=US_DL" \
  -H "Content-Type: image/jpeg" \
  -H "FileName: driving_license.jpg" \
  --data-binary "@/path/to/your/sample_driver_license.jpg"
```

### Expected Output
The application handles the asynchronous execution: uploading the file to IDP, waiting for a 10-second processing delay, polling the status, and returning a structured JSON payload:
```json
{
  "Address": "123 ELM ST, APARTMENT 4B, BOSTON MA 02116",
  "FirstName": "JOHN",
  "LastName": "DOE",
  "LicenseNumber": "S12345678"
}
```

---

## 🔄 Data Mapping & Transformation

The final step in the Mule flow normalizes the complex, nested response returned by the IDP service. Using DataWeave, it maps all configured field values into a clean, flat JSON object.

### DataWeave Mapping Code
```dw
%dw 2.0
output application/json
---
payload.fields mapObject ((value, key, index) -> (key):(value.value))
```
