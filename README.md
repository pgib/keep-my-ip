### Background

I have a server at home that I use for various things. My IP address, while generally stable, does change from time-to-time.

I run this script in a cron job on my server to keep a Route 53 record current.

I also have an EC2 instance with a security group that grants SSH access from my home. When my IP address changes, I don't want to be manually updating the ingress rules, so this script also takes care of that for me.

This is a very specific problem to solve, and I imagine this is only useful to me; however, in the event you have similar needs, feel free to use this.

### Prerequesites

Whatever user runs this script should have AWS credentials already set up either in the environment or in the `~/.aws/credentials` file. 

* A recent version of Ruby (I'm using 2.7.x at time of writing) and [Bundler](https://bundler.io/) installed.
* For an update to the Route 53 zone, you will need to know your hosted zone's ID.
* For an update to the security group's ingress rules, you will need to know your security group's ID.
* A URL that when requested will return your IP address (and only your IP address). `https://api.ipify.org?format=text` seems to be such a URL; however, I use Nginx with the [echo module](https://www.nginx.com/resources/wiki/modules/echo/) activated. See below for an example configuration block you can put into an Nginx config.

* Following best practices, the user running this should be limited to the following policy template:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ChangeRecordSet",
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/<MY ZONE ID HERE>",
            ]
        },
        {
            "Sid": "ListZonesAndGetChange",
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AuthorizeSecurityGroupIngress",
            "Effect": "Allow",
            "Action": "ec2:AuthorizeSecurityGroupIngress",
            "Resource": "arn:aws:ec2:*:*:security-group/<MY SECURITY GROUP ID>"
        },
        {
            "Sid": "DescribeSecurityGroups",
            "Effect": "Allow",
            "Action": "ec2:DescribeSecurityGroups",
            "Resource": "*"
        }
    ]
}
```

### Usage

The `--security-group-id` argument is optional. Everything is necessary.

```sh
./keep-my-ip --zone-id MYZONEID \
  --record-name home \
  --ip-url 'https://api.ipify.org?format=text' \
  --security-group-id sg-ABC123
```

### Nginx config

If you don't want to rely on a third-party service to return your URL and you happen to run an Nginx server external to your home, you can add the following block into a `server { ... }` context once you have activated the [echo module](https://www.nginx.com/resources/wiki/modules/echo/):

```nginx
  # what is my ip?
  location /ip
  {
    default_type text/plain;
    echo $remote_addr;
  }
```


