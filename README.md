# Integrate SNS with SQS  
  
  
SNS and SQS are managed services of AWS for pub-sub messaging and queue messaging. Both of the services are serverless and are used to carry messages around. But both of them have a different purpose.   
  
SNS is short for Simple Notification Service. It is a pub-sub based service. Whenever a message is published to SNS, it can be sent to all subscribers. There are many services that can subscribe to it like SQS or Lambda. The complete list can be found here  
  
https://docs.aws.amazon.com/sns/latest/dg/sns-create-subscribe-endpoint-to-topic.html  
  
We’ll use AWS SAM to build our infrastructure.   
  
We’ll make a service that sends emails to users of an e-commerce application whenever a customer makes a purchase.   
  
The architecture of our demo is below.  
  ![Lambda output](https://drive.google.com/uc?id=17PjQeozxYlin5rkvkilYiOjPgyp1R3ti)
  
Whenever a purchase is made, a message gets published to the SNS topic. It fans out the message to SQS. Let’s assume there is a lambda function that pulls messages from SQS and sends the emails. Everything works great. The code setup is below.  
  

      SnsToSqsPolicy:  
        Type: AWS::SQS::QueuePolicy  
        Properties:  
          PolicyDocument:  
            Version: "2012-10-17"  
            Statement:  
              - Sid: "Allow SNS publish to SQS"  
                Effect: Allow  
                Principal:  
                  Service: "sns.amazonaws.com"  
                Resource: !GetAtt SqsQueue.Arn  
                Action: SQS:SendMessage  
                Condition:  
                  ArnEquals:  
                    aws:SourceArn: !Ref SnsTopic  
          Queues:  
            - Ref: SqsQueue  
      
      SqsQueue:  
        Type: AWS::SQS::Queue  
        Properties:  
          QueueName: !Sub "${SqsQueueName}-${ENV}"  
      
      SnsTopic:  
        Type: AWS::SNS::Topic  
        Properties:  
          DisplayName: !Sub "${SNSName}-${ENV}"  
          Subscription:  
            - Protocol: sqs  
              Endpoint: !GetAtt SqsQueue.Arn  

  
  
Now there is a new requirement that we need to send a confirmation SMS also to our clients. One solution can be adding the SMS sending code in the lambda function that is responsible for sending emails. But by doing that we are bringing complexity to the system.   
  
One thing we can do is we can bring another queue and subscribe to it in the same SNS topic. A lambda then can pull messages from the second queue and we can code to send SMS from that branch. A nice and clean solution.   
  
By using this solution we can add more branches to process the same message and add new features without touching existing functions.   
  
After adding the second queue the architecture looks like this  
.  

![Lambda output](https://drive.google.com/uc?id=1BOUuK4LxiYquCEm0r1NyROdq-zfjo0yT)
  
The updated code looks like this  
  

    SnsToSqsPolicy:  
      Type: AWS::SQS::QueuePolicy  
      Properties:  
        PolicyDocument:  
          Version: "2012-10-17"  
      Statement:  
            - Sid: "Allow SNS publish to SQS"  
      Effect: Allow  
              Principal:  
                Service: "sns.amazonaws.com"  
      Resource: [  
                !GetAtt SqsQueue.Arn,  
                !GetAtt SqsQueue2.Arn,  
              ]  
              Action: SQS:SendMessage  
              Condition:  
                ArnEquals:  
                  aws:SourceArn: !Ref SnsTopic  
        Queues:  
          - Ref: SqsQueue  
          - Ref: SqsQueue2  
      
    SqsQueue:  
      Type: AWS::SQS::Queue  
      Properties:  
        QueueName: !Sub "${SqsQueueName}-${ENV}"  
      
    SqsQueue2:  
      Type: AWS::SQS::Queue  
      Properties:  
        QueueName: !Sub "${SqsQueueName}2-${ENV}"  
      
    SnsTopic:  
      Type: AWS::SNS::Topic  
      Properties:  
        DisplayName: !Sub "${SNSName}-${ENV}"  
      Subscription:  
          - Protocol: sqs  
            Endpoint: !GetAtt SqsQueue.Arn  
          - Protocol: sqs  
            Endpoint: !GetAtt SqsQueue2.Arn

  
Our next topic will be to integrate SQS to lambda.

You can get the source code in the following link  
  
https://github.com/masudur-rahman-niloy/SNS-SQS-integration
