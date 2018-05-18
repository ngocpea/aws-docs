

#### AWS - 180504 Realtime assumable IAM roles listing

------

##### Use case: Confluence page where the list of assumable roles is manually updated therefore can often be outdated

##### Action: Generate list of assumable IAM roles  to replace manually updated list

**Services**: EC2, IAM, STS, AWS-SDK (Ruby), ALB

------

#### create an ec2 instance

Things to note: 

— Tagging: 

​      Refer to https://github.com/cultureamp/sensible-defaults/blob/master/tagging.md

In addition, add tag `Name: {instance_name}`

— Security group: 

​      HTTP from VPN (port 80)

#### create and attach load balancer

Things to note: 

— Name: {same as ec2 instance name}-ALB

— Tagging:

​	Refer to: https://github.com/cultureamp/sensible-defaults/blob/master/tagging.md

In addition, add tag `Name: {ALB_name}`

— Security group: 

​      HTTP port 80 from **SG-id** of ec2 instance created above.

— Register target

#### on the ec2 instance

###### install apache

— install: `$ sudo yum -y install httpd`

— start service: `$ sudo service httpd start`

  for more information, refer to doc: https://docs.aws.amazon.com/efs/latest/ug/wt2-apache-web-server.html#wt2-apache-web-server-one-ec2-host

###### install ruby aws-sdk

— install command: `gem install aws-sdk` or `sudo gem install aws-sdk`

  https://docs.aws.amazon.com/sdk-for-ruby/v3/developer-guide/setup-install.html

— create folder and add two files 

1.   `list_iam_roles.rb`
2.   `iam_template.erb`

```
# list_iam_roles.rb

require 'aws-sdk-iam' # v2: require 'aws-sdk'
require 'json'
require 'erb'

def get_role_credential(role_arn)
  role_credentials = Aws::AssumeRoleCredentials.new(
    client: Aws::STS::Client.new(region: 'us-west-2'),
    role_arn: role_arn,
    role_session_name: 'role_session'
  )
return role_credentials
end

def print_roles(role_credentials)
  iam = Aws::IAM::Client.new(region: 'us-west-2', credentials: role_credentials)

  output = ''

  account_name = iam.list_account_aliases.account_aliases.first

  iam.list_roles.roles.each do |roles|
    policy = roles.assume_role_policy_document
    policy_json = JSON.parse(CGI.unescape(policy))

  aws_service = policy_json['Statement'][0]['Principal']['AWS']

  switch_role_url = ('https://signin.aws.amazon.com/switchrole?roleName=' + roles.role_name.tr(' ', '-').to_s + '&account=' + account_name.to_s).gsub(URI.regexp, '<a href="\0">\0</a>')

    if aws_service == 'arn:aws:iam::359618605424:root'
      output.concat("<tr><tbody><td>#{roles.role_name}</td><td> #{switch_role_url} </td><td>#{account_name}</td>")
    end
  end
return output
end

def print()
  iam_assumable_roles_arn = [
    'arn:aws:iam::700574780415:role/list-iam-roles-for-campers',
    'arn:aws:iam::527100417633:role/list-iam-roles-for-campers',
    'arn:aws:iam::529105607725:role/list-iam-roles-for-campers',
    'arn:aws:iam::904324412205:role/list-iam-roles-for-campers',
    'arn:aws:iam::226140413739:role/list-iam-roles-for-campers',
    'arn:aws:iam::359618605424:role/list-iam-roles-for-campers',
    'arn:aws:iam::514571838450:role/list-iam-roles-for-campers',
    'arn:aws:iam::082134150143:role/list-iam-roles-for-campers'
]

  iam_assumable_roles_arn.map do |item|
    print_roles(get_role_credential(item))
  end.join('')
end

# erb template
html = ERB.new(File.read('/home/ec2-user/aws-sdk/iam_template.erb')).result(binding)
puts html
```

```
# iam_template.erb

<!DOCTYPE html>
<html>
  <head>
   <title>IAM Roles</title>
  <meta name='viewport' content='width=device-width, initial-scale=1'>
  <link rel='stylesheet' href='https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css'>
  <script src='https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js'></script>
  <script src='https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js'></script>
  </head>
  <body>
    <div class='container'>
      <h2> IAM Roles </h2>
    <table class='table'>
    <thead>
      <tr>
        <th>Role Name</th>
        <th>URL</th>
        <th>Account Name</th>
      </tr>
    </thead>
    <tbody>
      <%= print() %>
    </tr>
    <tbody>
   </table>
    </div>
    </table>
  </body>
</html>
```

— install cronjob: `0 * * * * ruby /home/ec2-user/aws-sdk/iam_list_iam.rb > /var/www/html/index.html` to output results into html page to be viewed once a day

#### in the aws console

###### account(s) to cross into

— sign into each account that you'd like to retrieve information from

— create role eg. `list-iam-roles-for-campers` /*campers being the account ec2 lives in*/

​	— which includes explicit trust relationship of account the ec2 lives in

 	— include inline policy that retrieves iam information to generate `Role name`, `URL assumable link` and `Account name`:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:ListRoles",
                "iam:GetRole"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:ListAccountAliases",
            "Resource": "*"
        }
    ]
}
```

###### Camper's account

— create role `list-iam-roles-for-ec2`

— include inline policy: 

*allows for ec2 instance to `STS:AssumeRole` into another account

```
{
	"Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
            	"arn:aws:iam::529105607725:role/list-iam-role-for-campers", #add role arn as needed
            	"arn:aws:iam::527100417633:role/list-iam-role-for-campers" 
            ]
        }
    ]
}
```

 — add resource arn as needed

