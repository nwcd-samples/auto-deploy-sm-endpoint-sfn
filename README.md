# 使用AWS Step Functions在Sagemaker中持续部署TensorFlow模型


在 EC2 中执行 TensorFlow 模型训练，上传训练好的模型文件到 S3，在 Sagemaker 中进行部署，部署完成后得到一个 Sagemaker Endpoint，调用 Endpoint 进行预测。

对于定时 retrain 模型的场景，比如每天 retrain 一次 TensorFlow 模型，需要自动进行重新部署并且更新Sagemaker Endpoint，并保持 endpoint 不变，更新的过程中 Endpoint 需保持可用。本文基于此需求使用 Step Functions Data Science SDK 开发了一个在 Sagemaker 中执行 TensorFlow 模型自动部署的方案。



## 提前准备

1）确保Sagemaker Notebook具有S3和Step Functions权限。

创建给Sagemaker Notebook使用的IAM role，名为“AmazonSageMaker-ExecutionRole-StepFunctions”，赋予 “AWSStepFunctionsFullAccess”, “AmazonS3FullAcces”两个权限Policy，在创建Notebook的时候选择该Role，或者给已经创建好的Role添加权限。

2）确认Step functions具有Sagemaker权限

创建名为“ StepFunctionsWorkflowExecutionRole”的IAM role，并附加“ AmazonS3FullAccess” “ AmazonSageMakerFullAccess”两个策略。

3）确保进行模型训练的EC2具有Step Functions权限

本次调用状态机会直接在EC2服务器中通过python 脚本调用，因此需要确保EC2具有执行 Step Functions 的权限，可以给EC2 附加一个角色，给角色添加 “StepFunctionsFullAceess” Policy，或者给EC2中配置了AKSK的IAM User附加“StepFunctionsFullAceess” Policy。

 

 ## 创建 Step Functions 状态机

进入Sagemaker Console，创建笔记本实例，在 IAM 角色处选择上一步创建好的 IAM Role “AmazonSageMaker-ExecutionRole-StepFunctions”，或者给已经创建的角色附加 “AWSStepFunctionsFullAccess” 策略。

创建好笔记本实例后选择 “打开JupyterLab”，打开Terminal。

在终端中执行以下命令：

```
cd SageMaker
 wget https://becky-open-data.s3-ap-northeast-1.amazonaws.com/auto_deploy_sm_endpoint_sfn.ipynb
```

 打开名为 “auto_deploy_sm_endpoint_sfn”的笔记本，依次执行每个单元格，记得替换笔记本中的S3 bucket name 以及 workflow_execution_role 的ARN。

执行完成后，进入 Step Fucntions Console，可以看到创建好了一个状态机，点击进入。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image003.png)

 点击 “定义”，可以看到状态机的定义，左侧是生成的配置项，右侧是状态机工作流的图像呈现。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image004.png)

 

## 执行状态机

 在EC2中下载执行状态机的python脚本，并确保EC2安装了boto3的SDK。

```
pip install boto3

wget https://becky-open-data.s3-ap-northeast-1.amazonaws.com/excute_auto_deploy_sm_model_workflow.py
```

 修改脚本中 Step Functions 的 ARN 为你自己的ARN，可以在上传模型文件到S3成功后运行该python脚本，每次执行都会调用一次Step Functions状态机，并传入参数。每一次运行完后可以看到状态机下面有一条执行记录。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image005.png)

 

## 查看执行结果

点击执行，可以看到各个步骤的执行状态，以下是第一次执行该状态机的结果，第一次执行时需要create endpoint。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image005.png)

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image006.png)

以下是第二次执行该状态机的结果，第二次执行时需要update endpoint。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image007.png)

 

执行完成后可以前往 Sagemaker Console查看状态机的输出，可以看到保存的模型。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image008.png)

 

以及对应模型响应的终端节点配置。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image009.png)

 

以及对应的Endpoint，模型更新后保持同一个Endpoint不变。

![img](file:////Users/beibeizh/Library/Group%20Containers/UBF8T346G9.Office/TemporaryItems/msohtmlclip/clip_image010.png)

 

 ## 参考链接

- AWS Step Functions 开发者文档：https://docs.aws.amazon.com/zh_cn/step-functions/latest/dg/welcome.html
- Step functions python sdk: https://aws-step-functions-data-science-sdk.readthedocs.io/en/latest/sagemaker.html
- AWS Sagemaker 模型部署开发者文档：https://docs.aws.amazon.com/zh_cn/sagemaker/latest/dg/how-it-works-deployment.html#how-it-works-hosting
- Sagemaker python sdk: https://sagemaker.readthedocs.io/en/stable/frameworks/tensorflow/index.html
- Boto3 sdk: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sagemaker.html