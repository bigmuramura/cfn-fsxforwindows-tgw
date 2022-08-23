---
AWSTemplateFormatVersion: "2010-09-09"
Description: Creates the Amazon FSx for Windows File Server.

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: Common Settings
        Parameters:
          - ProjectName
          - Environment

Parameters:
  ProjectName:
    Description: Project Name
    Type: String
    Default: unnamed
  Environment:
    Description: Environment
    Type: String
    Default: dev
    AllowedValues:
      - prod
      - dev
      - stg
  S3BucketName:
    Description: Type of this BacketName.
    Type: String
  VPCID:
    Type: AWS::EC2::VPC::Id

Resources:
    # ------------------------------------------------------------#
    # Kinesis Data Firehose
    # ------------------------------------------------------------#
    KinesisFirehoseDeliveryStream:
      Type: "AWS::KinesisFirehose::DeliveryStream"
      Properties:
        DeliveryStreamName: !Sub "aws-fsx-${ProjectName}-${Environment}-auditlog-fsx"
        DeliveryStreamType: "DirectPut"
        DeliveryStreamEncryptionConfigurationInput:
          KeyType: "AWS_OWNED_CMK"
        ExtendedS3DestinationConfiguration:
          BucketARN: !GetAtt S3Bucket.Arn
          BufferingHints:
            SizeInMBs: 5
            IntervalInSeconds: 300
          CloudWatchLoggingOptions:
            Enabled: true
            LogGroupName: !Ref CloudWatchLogsLogGroup
            LogStreamName: !Ref CloudWatchLogsLogStream
          CompressionFormat: "UNCOMPRESSED"
          DataFormatConversionConfiguration:
            Enabled: false
          EncryptionConfiguration:
            NoEncryptionConfig: "NoEncryption"
          Prefix: ""
          RoleARN: !GetAtt KinesisIAMRole1.Arn
          ProcessingConfiguration:
            Enabled: false
          S3BackupMode: "Disabled"

  # --------------------------------------------
  # IAM Role
  # --------------------------------------------
    KinesisIAMRole1:
      Type: "AWS::IAM::Role"
      Properties:
        Path: "/service-role/"
        RoleName: !Sub "${ProjectName}-${Environment}-kinesisrole-fsx"
        AssumeRolePolicyDocument:
          Version: 2012-10-17
          Statement:
            - Effect: Allow
              Principal:
                Service: firehose.amazonaws.com
              Action: sts:AssumeRole
        MaxSessionDuration: 3600
        ManagedPolicyArns:
          - arn:aws:iam::aws:policy/AmazonElasticFileSystemFullAccess
          # - !Ref KinesisIAMPolicy1

      # ------------------------------------------------------------#
      # CloudWatch Logs
      # ------------------------------------------------------------#
    CloudWatchLogsLogGroup:
      Type: AWS::Logs::LogGroup
      Properties:
        LogGroupName: !Sub "/aws/kinesisfirehose/aws-fsx-${ProjectName}-${Environment}-auditlog-fsx"
        RetentionInDays: 90
    CloudWatchLogsLogStream:
      Type: AWS::Logs::LogStream
      Properties:
        LogGroupName: !Ref CloudWatchLogsLogGroup
        LogStreamName: "S3Delivery"

    # ------------------------------------------------------------------------------------ #
    # S3
    # ------------------------------------------------------------------------------------ #
    S3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: !Sub ${ProjectName}-${Environment}-${S3BucketName}-${AWS::AccountId}
        OwnershipControls:
          Rules:
            - ObjectOwnership: "BucketOwnerEnforced"
        PublicAccessBlockConfiguration:
          BlockPublicAcls: True
          BlockPublicPolicy: True
          IgnorePublicAcls: True
          RestrictPublicBuckets: True
        VersioningConfiguration:
          Status: Enabled
        BucketEncryption:
          ServerSideEncryptionConfiguration:
            - ServerSideEncryptionByDefault:
                SSEAlgorithm: "AES256"
              BucketKeyEnabled: false
        LifecycleConfiguration:
          Rules:
            - Id: AbortIncompleMultipartUpload # 未完了なマルチパートアップロードの削除
              AbortIncompleteMultipartUpload:
                DaysAfterInitiation: 7
              Status: "Enabled"