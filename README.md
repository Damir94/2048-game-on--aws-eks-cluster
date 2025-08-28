![Screenshot](https://github.com/user-attachments/assets/eba724e3-66b4-42d0-bd2a-622bb3a684cc)

Today we shall have a step-by-step end-to-end deployment of the 2048 game on the AWS EKS platform.

Prerequisites
 - Familiarity with K8S and AWS
 - kubectl — install kubectl command line tool for working with K8S clusters on your PC
 - eksctl — install this command line tool for working with EKS clusters and is very useful in automating individual tasks
 - AWS CLI — command line tool for working with AWS services.

Installation of these tools are beyond the scope of this article. Depending on your the platform you use, you can download and install the tools from the links above.

Setting up your AWS IAM users

 - Go to the IAM (Identity and Access Management) service in the AWS Management Console.
 - Click on “Users” in the left-hand navigation pane and then click on “Add user.”
 - Enter a username for the new IAM user and select the access type (Programmatic access, AWS Management Console access, or both).
 - Choose the permissions for the IAM user by adding them to one or more IAM groups or attaching policies directly. You can use admin permissions for lab purposes but I do emphasize on the least privilege assignment.
 - If you selected “Programmatic access” during user creation, you will receive access keys (Access Key ID and Secret Access Key).
 - Store these access keys securely, as they will be used to authenticate API requests made to AWS services.
