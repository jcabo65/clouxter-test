Clouxter – Infra con CloudFormation + GitHub Actions (Prueba Técnica)

Este repo despliega, en us-east-1, una infraestructura mínima con CloudFormation y GitHub Actions:

VPC con subnets públicas/privadas y route tables
S3 (bucket)
DynamoDB (tabla)
EC2 (instancia demo)
Site-to-Site VPN: VGW + CGW + VPNConnection con rutas estáticas hacia on-prem
Nota: Para pruebas, la VPN usa un IP de cliente de prueba (203.0.113.10) de modo que el stack se cree sin un equipo on-prem real. Los túneles saldrán DOWN (esperado en prueba).


Estructura del repositorio

├─ vpc-subnets.yaml            # Crea VPC, subnets y route tables (con Outputs)
├─ ec2-instances.yaml          # Instancia(s) EC2 demo (usa Outputs de la VPC)
├─ s3-bucket.yaml              # Bucket S3
├─ dynamodb-table.yaml         # Tabla DynamoDB
├─ vpn.yaml                    # VGW+CGW+VPNConnection (+ rutas estáticas)
└─ .github/
   └─ workflows/
      └─ deploy.yaml           # Pipeline que valida y despliega todo

Notas clave
vpn.yaml incluye DependsOn: AttachVpnGateway en VpnConnection y en todas las rutas para asegurar el orden: primero se adjunta el VGW a la VPC, luego se crean conexión y rutas (evita el error “gateway ID … does not exist”).
CustomerGatewayIp tiene Default: "203.0.113.10" para que la prueba pase sin secretos.

¿Qué hace el pipeline?

Archivo: .github/workflows/deploy.yaml
Se ejecuta en push a main cuando cambian: vpc-subnets.yaml, ec2-instances.yaml, dynamodb-table.yaml, s3-bucket.yaml, vpn.yaml.

Pasos:
Validate de cada plantilla con aws cloudformation validate-template.
Deploy VPC → captura VpcId, PrivateRouteTable1Id (PRT1), PrivateRouteTable2Id (PRT2), PublicSubnet1Id, PublicSubnet2Id en variables de entorno del job.
Deploy S3 y Deploy DynamoDB.
Deploy EC2 (usa Outputs de VPC).
Deploy VPN (usa VpcId, PRT1, PRT2).
No pasa CustomerGatewayIp → CloudFormation usa el Default del template.
Anti-regresiones opcionales (incluidos en mi ejemplo previo):
Assert required envs for VPN para fallar si VPC_ID, PRT1, PRT2 vienen vacíos.


Detalle del template de VPN (vpn.yaml)

| Parámetro                   | Tipo    | Default                                           | Descripción                                                      |
| --------------------------- | ------- | ------------------------------------------------- | ---------------------------------------------------------------- |
| `VpcId`                     | VPC::Id | —                                                 | VPC donde se adjunta el VGW                                      |
| `PrivateRouteTable1Id`      | String  | —                                                 | Route table privada (AZ1)                                        |
| `PrivateRouteTable2Id`      | String  | —                                                 | Route table privada (AZ2)                                        |
| `CustomerGatewayIp`         | String  | `203.0.113.10`                                    | IPv4 público del CGW (para prueba usa el default)                |
| `CustomerBgpAsn`            | Number  | `65000`                                           | ASN del lado cliente (requerido aunque uses rutas estáticas)     |
| `ExistingVpnGatewayId`      | String  | `""`                                              | Si lo pasas, reutiliza un VGW existente                          |
| `ExistingCustomerGatewayId` | String  | `""`                                              | Si lo pasas, reutiliza un CGW existente                          |
| `OnPremCidrA/B/C`           | String  | `172.16.0.0/16`, `172.16.1.0/24`, `172.16.2.0/24` | Rutas on-prem que se anuncian por la VPN                         |
| `CreateRoutes`              | String  | `true`                                            | Si `false`, no crea rutas hacia VGW en las route tables privadas |


Verificación (CLI)
REGION=us-east-1

# Stack y outputs de la VPN
aws cloudformation describe-stacks \
  --stack-name ClouxterVPNStack --region $REGION \
  --query "Stacks[0].[StackStatus,Outputs]" --output table

# Conexión VPN (tag Name=ClouxterVPN)
aws ec2 describe-vpn-connections \
  --region $REGION \
  --filters Name=tag:Name,Values=ClouxterVPN \
  --query "VpnConnections[0].{Id:VpnConnectionId,State:State,VgwTelemetry:VgwTelemetry[*].[OutsideIpAddress,Status,LastStatusChange,StatusMessage]}" \
  --output table

# VGW y attachment a la VPC
aws ec2 describe-vpn-gateways \
  --region $REGION \
  --filters Name=tag:Name,Values=ClouxterVGW \
  --query "VpnGateways[0].{Id:VpnGatewayId,State:State,Attachments:Attachments}" \
  --output table

# Rutas en las route tables privadas
aws ec2 describe-route-tables \
  --region $REGION \
  --route-table-ids $PRT1 $PRT2 \
  --query "RouteTables[].Routes[?GatewayId!=null].[DestinationCidrBlock,GatewayId,State]" \
  --output table

Resultado esperado (prueba):

Stack VPN: CREATE_COMPLETE
VPNConnection: State = available (túneles DOWN)
VGW: attached a tu VPC
Rutas privadas → GatewayId = vgw-… y State = active


