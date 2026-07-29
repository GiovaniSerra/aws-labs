import json
import boto3
import os
from botocore.exceptions import ClientError

client = boto3.client('events')

def lambda_handler(event, context):
  try:
    detail = event["detail"]
    detail["item"]["eventtype"]="delivered"
    response = client.put_events(
      Entries=[
        {
          'DetailType': 'eventtype',
          'Detail': json.dumps(detail),
          'EventBusName': os.environ.get('EVENT_BUS'),
          'Source':"deliver_pizza"
        },
      ]
    )
  except ClientError as err:
    print(err)
