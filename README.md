# CloudFormation Weekend Bootcamp

**De cero a Payment Platform Serverless en un fin de semana.**

Aprendiendo CloudFormation construyendo una plataforma de pagos serverless con arquitectura hexagonal, patrones DDD, Saga Pattern y alta disponibilidad.

## Objetivo

Desplegar una app que simule pagos por **Datafono / Link / Botón Web / QR** implementando:

- Arquitectura Hexagonal (Ports & Adapters)
- Principios SOLID
- Saga Pattern (orquestación con compensación)
- Repository Pattern
- Domain Events
- Alta Disponibilidad (Multi-Region)

## Stack Tecnológico

| Servicio | Uso |
|---|---|
| DynamoDB | Single-table design para pagos |
| Lambda (Python 3.12) | Compute serverless |
| API Gateway v2 | HTTP API |
| SQS + DLQ | Command Bus (desacoplamiento) |
| EventBridge | Domain Events |
| Step Functions | Saga Pattern (orquestación) |
| Route 53 + Global Tables | Alta disponibilidad multi-region |
| CloudWatch | Observabilidad |
| CloudFormation | IaC — todo se despliega con un comando |

## Ejercicios

| # | Ejercicio | Conceptos CF | Estado |
|---|---|---|---|
| 1 | [DynamoDB Table](#ejercicio-1--dynamodb-table) | `AWS::DynamoDB::Table`, Parameters, Outputs, Export | ✅ |
| 2 | [Lambda + IAM Role](#ejercicio-2--lambda--iam-role) | `AWS::Lambda::Function`, `AWS::IAM::Role`, `!ImportValue`, `!GetAtt` | ✅ |
| 3 | [API Gateway HTTP API](#ejercicio-3--api-gateway-http-api) | `AWS::ApiGatewayV2::Api`, `!Sub`, Lambda Permission | ⬜ |
| 4 | [S3 + Empaquetado real](#ejercicio-4--s3--empaquetado-real) | `cloudformation package`, S3 artifacts, DeletionPolicy | ⬜ |
| 5 | [SQS + Dead Letter Queue](#ejercicio-5--sqs--dead-letter-queue) | `AWS::SQS::Queue`, RedrivePolicy, EventSourceMapping | ⬜ |
| 6 | [EventBridge](#ejercicio-6--eventbridge) | `AWS::Events::EventBus`, EventPattern, Rules | ⬜ |
| 7 | [Step Functions — Saga](#ejercicio-7--step-functions-saga-pattern) | `AWS::StepFunctions::StateMachine`, ASL, Catch/Compensate | ⬜ |
| 8 | [Hexagonal Architecture](#ejercicio-8--hexagonal-architecture) | Code structure, DDD, Ports/Adapters (no CF nuevo) | ⬜ |
| 9 | [Nested Stacks](#ejercicio-9--nested-stacks) | `AWS::CloudFormation::Stack`, composición, `!GetAtt Stack.Outputs` | ⬜ |
| 10 | [Multi-Region HA + Observabilidad](#ejercicio-10--multi-region-ha--observabilidad) | Global Tables, Route53 Failover, CloudWatch Alarms | ⬜ |

## Estructura del Proyecto

```
payment-platform/
├── root-stack.yaml                 ← Ej 9: orquestador de nested stacks
├── infra/
│   ├── 01-dynamodb.yaml            ← Ej 1: tabla de pagos
│   ├── 04-artifacts-bucket.yaml    ← Ej 4: bucket para code artifacts
│   ├── 05-sqs.yaml                 ← Ej 5: colas + DLQ
│   └── 06-eventbridge.yaml         ← Ej 6: event bus
├── compute/
│   ├── 02-lambda-role.yaml         ← Ej 2: Lambda + IAM
│   ├── 07-step-functions.yaml      ← Ej 7: Saga orchestration
│   └── src/
│       └── create-payment/
│           ├── index.py            ← Entry point (adapter HTTP)
│           ├── domain/
│           │   ├── entities/
│           │   │   └── payment.py
│           │   └── value_objects/
│           │       └── payment_method.py   # DATAFONO|LINK|QR|BOTON_WEB
│           ├── ports/
│           │   └── payment_repository.py   # Interface
│           ├── adapters/
│           │   └── dynamodb_payment_repository.py
│           └── application/
│               └── create_payment_use_case.py
├── api/
│   └── 03-api-gateway.yaml         ← Ej 3: HTTP API
├── ha/
│   └── 10-multi-region.yaml        ← Ej 10: HA + monitoring
└── observability/
    └── monitoring.yaml              ← Ej 10: alarms + dashboard
```

## Ejercicio 1 — DynamoDB Table

**Objetivo:** Primer deploy. Un recurso, un output.

**Conceptos aprendidos:**
- Estructura de un template CF: `Parameters`, `Resources`, `Outputs`
- `!Ref` devuelve el nombre/ID de un recurso
- `!GetAtt Recurso.Atributo` obtiene propiedades específicas (como ARN)
- `!Sub "texto-${Variable}"` para interpolación de strings
- `Export/ImportValue` para cross-stack references
- DynamoDB: `AttributeDefinitions` declara atributos, `KeySchema` asigna roles (HASH/RANGE)
- Solo se declaran en `AttributeDefinitions` los atributos usados en keys/GSIs

**Desplegar:**
```bash
aws cloudformation validate-template --template-body file://infra/01-dynamodb.yaml

aws cloudformation deploy \
  --template-file infra/01-dynamodb.yaml \
  --stack-name endy-lab-dynamodb-dev \
  --parameter-overrides Env=dev
```

**Verificar:**
```bash
aws dynamodb describe-table --table-name endy-lab-payments-dev
aws cloudformation describe-stacks --stack-name endy-lab-dynamodb-dev
```

## Ejercicio 2 — Lambda + IAM Role

**Objetivo:** Crear una Lambda con su IAM Role y conectarla a DynamoDB vía cross-stack reference.

**Conceptos aprendidos:**
- Un IAM Role tiene 3 partes: Trust Policy (quién lo usa), Managed Policies (permisos precocinados), Inline Policies (permisos custom)
- `AssumeRolePolicyDocument` define quién puede asumir el role (`lambda.amazonaws.com`)
- `!ImportValue` trae exports de otros stacks — el nombre debe coincidir **exactamente**
- Para anidar funciones intrínsecas: una en forma corta (`!Sub`) y otra en forma larga (`Fn::ImportValue`)
- `--capabilities CAPABILITY_NAMED_IAM` es obligatorio cuando el template crea IAM Roles
- `!GetAtt Recurso.Arn` obtiene el ARN de un recurso del mismo template
- Lambda necesita `Environment.Variables` para recibir configuración (como TABLE_NAME)
- En macOS, usar `fileb://` para payloads de Lambda evita problemas con encoding

**Desplegar:**
```bash
aws cloudformation validate-template --template-body file://compute/02-lambda-role.yaml

aws cloudformation deploy \
  --template-file compute/02-lambda-role.yaml \
  --stack-name endy-lab-lambda-dev \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides Env=dev
```

**Verificar:**
```bash
printf '{"test": true}' > payload.json
aws lambda invoke \
  --function-name endy-lab-create-payment-dev \
  --payload fileb://payload.json \
  output.json && cat output.json
```

## Ejercicio 3 — API Gateway HTTP API

*Pendiente*

## Ejercicio 4 — S3 + Empaquetado real

*Pendiente*

## Ejercicio 5 — SQS + Dead Letter Queue

*Pendiente*

## Ejercicio 6 — EventBridge

*Pendiente*

## Ejercicio 7 — Step Functions: Saga Pattern

*Pendiente*

## Ejercicio 8 — Hexagonal Architecture

*Pendiente*

## Ejercicio 9 — Nested Stacks

*Pendiente*

## Ejercicio 10 — Multi-Region HA + Observabilidad

*Pendiente*

---

## Comandos CF que vas a repetir 1000 veces

```bash
# Validar antes de deploy (SIEMPRE)
aws cloudformation validate-template --template-body file://template.yaml

# Deploy (crea o actualiza automáticamente)
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name nombre-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides Env=dev

# Ver outputs
aws cloudformation describe-stacks --stack-name nombre-stack

# Debug cuando algo falla (tu mejor amigo)
aws cloudformation describe-stack-events --stack-name nombre-stack

# Borrar stack (limpia todo)
aws cloudformation delete-stack --stack-name nombre-stack
```

## Convenciones

- **Prefijo de stacks:** `endy-lab-{recurso}-{env}`
- **Tags en todo recurso:** `Owner: Endy`, `Project: Serverless-Workshop`, `Environment: Dev`
- **Región:** `ca-central-1`

## Limpieza

Después de cada sesión, borrar los stacks para evitar costos:

```bash
# Borrar en orden inverso (dependencias primero)
aws cloudformation delete-stack --stack-name endy-lab-api-dev
aws cloudformation delete-stack --stack-name endy-lab-lambda-dev
aws cloudformation delete-stack --stack-name endy-lab-dynamodb-dev
```

---

**Autor:** Endy B — Learning by doing, one stack at a time.