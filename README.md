# conta-banco-stepfunctions-aws
A ideia desse projeto é mostrar de forma simples um cadastro e a recuperação de dados.

# 📚 Workflow AWS Step Functions

Este projeto define um **workflow no AWS Step Functions** que integra múltiplos serviços da AWS, automatizando a criação de sessão no S3, execução de uma Lambda e ações paralelas envolvendo DynamoDB e S3 Glacier.

---

## 🚀 Fluxo do Workflow

1. **CreateSession (S3)**  
   Cria uma sessão no bucket do S3 (`MyData`).

2. **Lambda Invoke**  
   Executa a função Lambda `contaBanco`, passando como payload a entrada do estado anterior.  
   - Inclui política de **retry** em caso de falha (com `BackoffRate` e `JitterStrategy`).

3. **Paralelo**  
   Executa **duas tarefas em paralelo**:  
   - **DynamoDB UpdateItem**: Atualiza o item na tabela `MyDynamoDBTable`.  
   - **GetVaultAccessPolicy (Glacier)**: Recupera a política de acesso de um cofre no S3 Glacier.  

4. **End**  
   O workflow é finalizado após a execução bem-sucedida de ambos os ramos paralelos.

---

## 🖼️ Diagrama

![Step Functions Workflow](./stepfunctions_graph.png)

---

## 📑 Definição da State Machine (JSON)

```json
{
  "Comment": "A description of my state machine",
  "StartAt": "CreateSession",
  "States": {
    "CreateSession": {
      "Type": "Task",
      "Arguments": {
        "Bucket": "MyData"
      },
      "Resource": "arn:aws:states:::aws-sdk:s3:createSession",
      "Next": "Lambda Invoke"
    },
    "Lambda Invoke": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Output": "{% $states.result.Payload %}",
      "Arguments": {
        "FunctionName": "arn:aws:lambda:us-east-1:041616363742:function:contaBanco",
        "Payload": "{% $states.input %}"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException",
            "Lambda.TooManyRequestsException"
          ],
          "IntervalSeconds": 1,
          "MaxAttempts": 3,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "Next": "Paralelo"
    },
    "Paralelo": {
      "Type": "Parallel",
      "End": true,
      "Branches": [
        {
          "StartAt": "DynamoDB UpdateItem",
          "States": {
            "DynamoDB UpdateItem": {
              "Type": "Task",
              "Resource": "arn:aws:states:::dynamodb:updateItem",
              "Arguments": {
                "TableName": "MyDynamoDBTable",
                "Key": {
                  "Column": {
                    "S": "MyEntry"
                  }
                },
                "UpdateExpression": "SET MyKey = :myValueRef",
                "ExpressionAttributeValues": {
                  ":myValueRef": {
                    "S": "MyValue"
                  }
                }
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "GetVaultAccessPolicy",
          "States": {
            "GetVaultAccessPolicy": {
              "Type": "Task",
              "Arguments": {
                "AccountId": "MyData",
                "VaultName": "MyData"
              },
              "Resource": "arn:aws:states:::aws-sdk:glacier:getVaultAccessPolicy",
              "End": true
            }
          }
        }
      ]
    }
  },
  "QueryLanguage": "JSONata"
}
