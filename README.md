## Private Marketplace Automation
Private Marketplace automation is a set of simple templates that can be used to enable, create, and manage private marketplaces for your organization. 

For organizations that have software procurement policies and processes in place, Private Marketplace provides controls to ensure users are operating within those policies while using AWS Marketplace. Once enabled, users will only be able to procure products approved within their private marketplace. This ensures that only vetted products adhering to the organization’s policies can be purchased, helping reduce the risk of unapproved purchases.

Private Marketplace Automation uses AWS CloudFormation and provides templates to to enable the Private Marketplace feature in your organization, create and configure multiple private marketplace experiences, and manage these experiences. Internally, this solution uses AWS Marketplace Catalog APIs for private marketplace through AWS SDK for Python and leverages AWS Lambda, AWS EventBridge, and Amazon S3 for the end to end automation. 

## Prerequisites 
Before you begin, make sure you have access to the following:
1.	An AWS Organizations in all features enabled mode. 
       	
    a. Access to the management account to enable Private Marketplace feature.
    	
    b. An account in the organization to register as a delegated administrator for Private Marketplace. This is optional. If you do not register a delegated administrator, you can continue using the management account to configure and manage private marketplace experiences.
2.	Permissions to create and update AWS CloudFormation stacks from the management and delegated administrator accounts.


## Architecture

The architecture for this solution is the following:
<br><br>
![Architecture](/Images/pmp-automation-architecture.png)
 
## Description

Private Marketplace Automation showcases the power of APIs and how they can be paired wih various AWS technologies to automate the setup of private marketplaces. It enables IT administrators to easily curate multiple private marketplaces for the entire organization, different organizational units, or individual accounts in their organization. 

This project contains [sample configuration files](./configuration_files) in JSON format that you can use to set up and manage private marketplaces. The creation and management is done using AWS Lambda that is triggered when files are uploaded to an Amazon S3 bucket. The status is monitored by another lambda that is triggered by an AWS EventBridge rule. You can also set up notifications using AWS User Notifications to get notified on EventBridge success and failure events.

## AWS Services and Features Used

* <b>[Private Marketplace](https://docs.aws.amazon.com/marketplace/latest/buyerguide/private-marketplace.html)</b> enables administrators to build customized digital catalogs of approved products from AWS Marketplace. Administrators can create unique sets of vetted software available in AWS Marketplace for an entire AWS Organization, AWS organizational units (OUs) or different AWS accounts within their organization.
* <b>[AWS Marketplace Catalog APIs for private marketplace](https://docs.aws.amazon.com/marketplace-catalog/latest/api-reference/private-marketplace.html)</b> provides an API interface to manage AWS Marketplace for your AWS organization. For private marketplace administrators, you can manage your private marketplace programmatically.
* <b>[Amazon S3](https://aws.amazon.com/s3/)</b> is an object storage service that offers industry-leading scalability, data availability, security, and performance.
* <b>[AWS Lambda](https://aws.amazon.com/lambda/)</b> lets you run code without provisioning or managing servers.
* <b>[AWS CloudFormation](https://aws.amazon.com/cloudformation/)</b> provides a common language for you to model and provision AWS and third party application resources in your cloud environment.
* <b>[Amazon EventBridge](https://aws.amazon.com/eventbridge/)</b> is a serverless event bus service that makes it easy to connect your applications with data from a variety of sources.
* <b>[AWS User Notifications](https://aws.amazon.com/notifications/)</b> is an AWS service that acts as a central location for your AWS notifications in the AWS Management Console.


## Solution overview
This solution enables you to automate Private Marketplace setup in your organization and performs the following steps: 

1.	Enable the Private Marketplace feature in your organization.
2.	Deploy Cloudformation stacks to set up AWS resources to create and manage private marketplace experiences.
3.	Upload configuration files to S3 to trigger creation or management of private marketplace experiences.
4.	Monitor the system to catch failures or confirm successful completion.



#### Creating AWS CloudFormation stack: 

1. Sign in to your AWS account and navigate to the [Create stack option in AWS CloudFormation console](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create).
2. Select the options Choose an <b>existing template</b> and Upload a template file. 
3. Select a template file, and specify stack details such as name and parameter values and choose default values for rest. 
4. On the final page, acknowledge that AWS CloudFormation might create IAM resources.
5. Choose Submit. Stack creation completes when you see a CREATE_COMPLETE status.
  
### Enable, create, manage and monitor private marketplace


#### Step 1: Enable the Private Marketplace feature in your organization


1. Deploy [enable_private_marketplace.yaml]( ./CloudFormation/enable_private_marketplace.yaml) template with <b>stack name</b>, such as "PMPEnablement".
2. To register delegated administrator, enter an AWS account id in the DelegatedAdministratorAccount parameter. This will allow you to perform the configuration and monitoring steps from the delegated administrator account. 

<b>Outcome: </br>
1.	Created of a service-linked role in the management account to describe AWS Organizations and update private marketplace resources.
2.	Enabled trusted access for AWS Organizations for your private marketplace as a trusted service.
3.	Optionally, registered a delegated administrator account.


#### Step 2: Configure and monitor private marketplace resources

If a delegated administrator is setup in step 1, private marketplace can be created from management account as well as delegated administrator account. 

1. Deploy [configure_private_marketplace.yaml.yaml](./CloudFormation/configure_private_marketplace.yaml) template with <b>stack name</b>, such as "PMPEnablement".

2. For parameter ExperiencesS3BucketName, choose an S3 bucket in the account. 

3. Choose defaults in next step and create stack. 

4. Deploy [private_marketplace_event_listener.yaml](./CloudFormation/private_marketplace_event_listener.yaml) template with <b>stack name</b>, such as "MonitorPMPStack"

5. For parameter ExperiencesS3BucketName, choose the same S3 bucket from step 2. 

6. Choose defaults in next step and create stack. 

<b>Outcome: </br>

ConfigurePMPStack stack sets up the following resources: 

1.	S3 bucket to upload experience configuration files and store status files.
2.	A lambda AsyncConfigurePrivateMarketplaceLambda that is triggered by S3 object put events. It reads the configuration file and starts change set to create or manage the experience.

MonitorPMPStack stack sets up the following resources.

1.	EventBridge rule to listen to change set status.
2.	ChangesetStatusUpdateLambda that gets triggered on change set status update. It writes to the status file and creates an error file if change set does not succeed.


#### Step 3: Upload configuration files to Amazon S3 bucket

It is recommended to start with a default private marketplace that is associated to the whole organization and customize or create more as you need them. 

1. The name for a configuration file should follow the format Experience_ShortName#version_id.json. 

    a.	ShortName must be unique and should be followed by a #. 

    b.	version_id  is optional. You can use it to audit the experience management.
2. Each configuration file can contain the following details for an experience. Refer schema [pmp_automation_json_schema.json](./configuration_files/pmp_automation_json_schema.json):

    a.	Name – Name of the experience. 
    
    b.	AssociatePrincipals – Principals can be a list of your organization ID, one or more OU IDs, or account IDs that will be associated to your private marketplace experience. If you specify principals that are associated to other experience, this will result in an error.  
    
    c.	DisassociatePrincipals – Same as above. These principals will be disassociated from your private marketplace experience. If you specify principals that are not associated to the experience, this will result in an error. 
    
    d.	AllowProducts – List of product IDs of the products to allow for procurement in the experience. To find the product IDs, refer to Finding products in the AWS Marketplace Catalog guide.
    
    e.	DenyProducts – List of product IDs of the products to deny for procurement in the experience. 
    
    f.	Status – Status of the experience. 
    
    g.	PolicyResourceRequests – Setting to allow or deny users to request for new products. 

#### Step 4: Monitoring the resources 

Once the updates are complete, the solution writes status back to the STATUS_ file in the S3 bucket. If there is a failure, an ERROR_ file will be written. Check the S3 bucket or set up additional monitoring, as required.

#### JSON schema for configuration files

Refer [pmp_automation_json_schema.json](./configuration_files/pmp_automation_json_schema.json) for schema of configuration files.


## Cleanup

1. Using AWS Console, select <b>‘CloudFormation’ </b>from the list of AWS Services.
1. Select the <b>Stacks </b>you created.
1. Click <b>‘Delete’ </b>action button to delete the stacks and all associated resources. 
1. Please make sure to delete your S3 bucket and CloudWatch logs


## Contributing

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

