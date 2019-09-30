## Custom Resource Helper

Simplify best practice Custom Resource creation, sending responses to CloudFormation and providing exception, timeout 
trapping, and detailed configurable logging.

[![PyPI Version](https://img.shields.io/pypi/v/crhelper.svg)](https://pypi.org/project/crhelper/)
![Python Versions](https://img.shields.io/pypi/pyversions/crhelper.svg)
[![Build Status](https://travis-ci.com/aws-cloudformation/custom-resource-helper.svg?branch=master)](https://travis-ci.com/aws-cloudformation/custom-resource-helper)
[![Test Coverage](https://img.shields.io/codecov/c/github/aws-cloudformation/custom-resource-helper.svg)](https://codecov.io/github/aws-cloudformation/custom-resource-helper)

## Features

* Dead simple to use, reduces the complexity of writing a CloudFormation custom resource
* Guarantees that CloudFormation will get a response even if an exception is raised
* Returns meaningful errors to CloudFormation Stack events in the case of a failure
* Polling enables run times longer than the lambda 15 minute limit
* JSON logging that includes request id's, stack id's and request type to assist in tracing logs relevant to a 
particular CloudFormation event
* Catches function timeouts and sends CloudFormation a failure response
 
## Installation

Install into the root folder of your lambda function

```json
cd my-lambda-function/
pip install crhelper -t .
```

## Example Usage

[This blog](https://aws.amazon.com/blogs/infrastructure-and-automation/aws-cloudformation-custom-resource-creation-with-python-aws-lambda-and-crhelper/) covers usage in more detail.

```python
from __future__ import print_function
from crhelper import CfnResource
import logging

logger = logging.getLogger(__name__)
# Initialise the helper, all inputs are optional, this example shows the defaults
helper = CfnResource(json_logging=False, log_level='DEBUG', boto_level='CRITICAL')

try:
    ## Init code goes here
    pass
except Exception as e:
    helper.init_failure(e)


@helper.create
def create(event, context):
    logger.info("Got Create")
    # Optionally return an ID that will be used for the resource PhysicalResourceId, 
    # if None is returned an ID will be generated. If a poll_create function is defined 
    # return value is placed into the poll event as event['CrHelperData']['PhysicalResourceId']
    #
    # To add response data update the helper.Data dict
    # If poll is enabled data is placed into poll event as event['CrHelperData']
    helper.Data.update({"test": "testdata"})

    # To return an error to cloudformation you raise an exception:
    if not helper.Data.get("test"):
        raise ValueError("this error will show in the cloudformation events log and console.")
    
    return "MyResourceId"


@helper.update
def update(event, context):
    logger.info("Got Update")
    # If the update resulted in a new resource being created, return an id for the new resource. CloudFormation will send
    # a delete event with the old id when stack update completes


@helper.delete
def delete(event, context):
    logger.info("Got Delete")
    # Delete never returns anything. Should not fail if the underlying resources are already deleted. Desired state.


@helper.poll_create
def poll_create(event, context):
    logger.info("Got create poll")
    # Return a resource id or True to indicate that creation is complete. if True is returned an id will be generated
    return True


def handler(event, context):
    helper(event, context)

```

### Polling

If you need longer than the max runtime of 15 minutes, you can enable polling by adding additional decorators for 
`poll_create`, `poll_update` or `poll_delete`. When a poll function is defined for `create`/`update`/`delete` the 
function will not send a response to CloudFormation and instead a CloudWatch Events schedule will be created to 
re-invoke the lambda function every 2 minutes. When the function is invoked the matching `@helper.poll_` function will 
be called, logic to check for completion should go here, if the function returns `None` then the schedule will run again 
in 2 minutes. Once complete either return a PhysicalResourceID or `True` to have one generated. The schedule will be 
deleted and a response sent back to CloudFormation. If you use polling the following additional IAM policy must be 
attached to the function's IAM role:

```yaml
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:AddPermission",
        "lambda:RemovePermission",
        "events:PutRule",
        "events:DeleteRule",
        "events:PutTargets",
        "events:RemoveTargets"
      ],
      "Resource": "*"
    }
  ]
}
```

## Credits

Decorator implementation inspired by https://github.com/ryansb/cfn-wrapper-python

Log implementation inspired by https://gitlab.com/hadrien/aws_lambda_logging

## License

This library is licensed under the Apache 2.0 License.
