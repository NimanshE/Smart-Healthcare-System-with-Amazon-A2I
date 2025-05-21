# Smart Healthcare System with Amazon Augmented AI

## Overview
The Smart Healthcare System is a cloud-native application that leverages Amazon Web Services (AWS) and Amazon Augmented AI (A2I) to automate medical document processing while integrating human-in-the-loop validation for critical healthcare decisions. This system addresses key challenges in modern healthcare, including processing large volumes of patient data, ensuring diagnostic accuracy, and optimizing medical expert resources.

## Features
- **Automated Document Processing**: Extract text and medical entities from uploaded documents using Amazon Textract and Comprehend Medical
- **Human-in-the-Loop Validation**: Route low-confidence detections to human experts via Amazon A2I workflows
- **Serverless Architecture**: Utilize AWS Lambda functions for scalable, event-driven processing
- **Secure Document Storage**: Manage document uploads and processed results in Amazon S3
- **RESTful API Interface**: Access the system through Amazon API Gateway endpoints
- **Web-Based Frontend**: Upload documents and view processing results through a simple web interface

## Architecture
![System Architecture](architecture.png)

The system follows a serverless architecture pattern with the following components:
1. **Storage Layer**: Amazon S3 buckets for input data and processed results
2. **Processing Layer**: AWS Lambda functions for document processing and workflow orchestration
3. **AI/ML Layer**: Amazon Textract and Comprehend Medical for document analysis
4. **Human Review Layer**: Amazon A2I for expert validation of uncertain results
5. **API Layer**: Amazon API Gateway for RESTful endpoints
6. **Monitoring Layer**: Amazon CloudWatch for logging and performance tracking
7. **Security Layer**: AWS IAM for role-based access control

## AWS Services Used
- **Amazon S3**: Storage for uploaded documents and processed results
- **AWS Lambda**: Serverless backend logic for processing and orchestrating workflows
- **Amazon Textract**: Text extraction from medical images and PDFs
- **Amazon Comprehend Medical**: Detection of medical entities from extracted text
- **Amazon Augmented AI (A2I)**: Human-in-the-loop review for low-confidence documents
- **Amazon API Gateway**: Public API endpoints to upload, list, and fetch documents
- **Amazon CloudWatch**: Logging and monitoring
- **Amazon IAM**: Secure role-based access to AWS services
- **Amazon SageMaker**: Used for creating human review workflows

## Setup and Deployment

### Prerequisites
- AWS Account with appropriate permissions
- AWS CLI configured locally
- Basic knowledge of AWS services

### Deployment Steps

1. **Set up S3 buckets**
   ```bash
   aws s3 mb s3://healthcare-input-data
   aws s3 mb s3://healthcare-processed-data-vacc
   aws s3 mb s3://healthcare-web-interface
   ```

2. **Configure S3 CORS for web uploads**
   ```bash
   aws s3api put-bucket-cors --bucket healthcare-input-data --cors-configuration file://cors-config.json
   ```

3. **Create IAM role for healthcare processing**
   ```bash
   aws iam create-role --role-name HealthcareProcessingRole --assume-role-policy-document file://lambda-trust-policy.json
   aws iam attach-role-policy --role-name HealthcareProcessingRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
   aws iam attach-role-policy --role-name HealthcareProcessingRole --policy-arn arn:aws:iam::aws:policy/ComprehendMedicalFullAccess
   aws iam attach-role-policy --role-name HealthcareProcessingRole --policy-arn arn:aws:iam::aws:policy/AmazonTextractFullAccess
   aws iam attach-role-policy --role-name HealthcareProcessingRole --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
   ```

4. **Deploy Lambda functions**
   ```bash
   aws lambda create-function --function-name GenerateUploadUrl --runtime python3.9 --role $ROLE_ARN --handler lambda_function.lambda_handler --zip-file fileb://generate_upload_url.zip
   aws lambda create-function --function-name DocumentProcessor --runtime python3.9 --role $ROLE_ARN --handler lambda_function.lambda_handler --zip-file fileb://document_processor.zip
   aws lambda create-function --function-name FetchResults --runtime python3.9 --role $ROLE_ARN --handler lambda_function.lambda_handler --zip-file fileb://fetch_results.zip
   aws lambda create-function --function-name DocumentDetails --runtime python3.9 --role $ROLE_ARN --handler lambda_function.lambda_handler --zip-file fileb://document_details.zip
   ```

5. **Set up S3 trigger for DocumentProcessor Lambda**
   ```bash
   aws lambda add-permission --function-name DocumentProcessor --statement-id s3-trigger --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::healthcare-input-data
   
   aws s3api put-bucket-notification-configuration --bucket healthcare-input-data --notification-configuration file://notification-config.json
   ```

6. **Create API Gateway**
   ```bash
   aws apigateway create-rest-api --name HealthcareSystemAPI
   ```
   Then create resources and methods for:
   - `/upload-url` (GET)
   - `/results` (GET)
   - `/document/{id}` (GET)

7. **Set up Amazon A2I workflow**
   ```bash
   # Create workforce
   aws sagemaker create-workforce --workforce-name "HealthcareReviewTeam" --cognito-config "UserPool=$USER_POOL_ID,ClientId=$CLIENT_ID"
   
   # Create human task UI
   aws sagemaker create-human-task-ui --human-task-ui-name "healthcare-review-ui" --ui-template "Content=$(cat worker-template.html | jq -Rs .)"
   
   # Create workteam
   aws sagemaker create-workteam --workteam-name HealthcareReviewTeam --member-definitions '[{"CognitoMemberDefinition": {"UserPool": "$USER_POOL_ID", "ClientId": "$CLIENT_ID", "UserGroup": "healthcare-reviewers"}}]' --description "Private team for healthcare document reviewers"
   
   # Create flow definition
   aws sagemaker create-flow-definition --flow-definition-name "healthcare-document-review" --role-arn $ROLE_ARN --human-loop-config file://human-loop-config.json --output-config '{"S3OutputPath": "s3://healthcare-processed-data-vacc/human-review/"}'
   ```

8. **Deploy web interface to S3**
   ```bash
   aws s3 sync ./web-interface s3://healthcare-web-interface --acl public-read
   ```

## API Endpoints

- **GET /upload-url**: Generate pre-signed URL for secure document upload
- **GET /results**: List processed documents with their status
- **GET /document/{id}**: Get detailed analysis results for a specific document

## Web Interface

The web interface is accessible at: http://healthcare-web-interface.s3-website-us-east-1.amazonaws.com/

Features:
- Upload medical documents
- View processing status
- View extracted medical entities and human review results

## Future Enhancements
- Integration of IoT medical device data streams
- Dashboard creation using Amazon QuickSight for better visualization
- Medical image analysis using Amazon Rekognition
- Enhanced error handling and retry mechanisms
- Multi-user authentication via Amazon Cognito

## Contributors
- Nimansh Endlay

## References
1. [Amazon Augmented AI Documentation](https://docs.aws.amazon.com/augmented-ai/)
2. [Build a Human in the Loop System for ML Models](https://aws.amazon.com/blogs/machine-learning/build-a-human-in-the-loop-workflow-for-machine-learning-models/)
3. [AWS Solutions for Healthcare](https://aws.amazon.com/health/solutions/)
4. Brannelly, R., Agarwal, M., Cardoz, A., Learning Amazon SageMaker, Packt Publishing, 2023.
5. Nagar, S., Madduri, A., "Integrating Human Intelligence with Artificial Intelligence for Healthcare Diagnostics," Journal of Cloud Computing Advances, 2022.
