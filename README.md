# Creación y acceso a un punto de enlace de AWS Client VPN mediante autenticación basada en Active Directory
**AWS Client VPN** es un servicio administrado de VPN que permite establecer conexiones <em>Host-to-Site</em> para acceder a los recursos de AWS (e Internet) a través de clientes OpenVPN.

AWS Client VPN permite autenticar a los usuarios bien mediante certificados de cliente (autenticación mutua), bien mediante un AD, o bien a través de un proveedor de identidades compatible con SAML2.0.

En este repositorio se muestra cómo se puede configurar un punto de enlace de AWS Client VPN mediante autenticación AD (<em>Active Directory</em>) y cómo realizar una conexión contra dicho punto de enlace.

![AWS Client VPN](/images/client-vpn.png)

## Requerimientos
* Disponer de el software `easyrsa` (https://github.com/OpenVPN/easy-rsa) instalado en la máquina cliente
* Disponer de una cuenta de AWS o de acceso a un sandbox de AWS Academy Learner Lab
* Disponer de un entorno Linux con acceso programático configurado a los servicios de AWS

## Instrucciones
1. Si no se ha realizado ya, es necesario crear una nueva PKI (<em>Public Key Infrastructure</em>) y una CA (<em>Certificate Authority</em>) para emitir certificados de confianza.
    
        /usr/share/easy-rsa/easyrsa init-pki
        /usr/share/easy-rsa/easyrsa build-ca nopass
        
    Completar el proceso indicando el <em>Common Name</em>
    
2. Generar las claves RSA privada y pública para el servicio de AWS Client VPN:

        /usr/share/easy-rsa/easyrsa build-server-full server nopass
          
3. Organizar la estructura de los certificados y claves generados:

        mkdir ~/certs/
        cp pki/ca.crt ~/certs/
        cp pki/issued/server.crt ~/certs/
        cp pki/private/server.key ~/certs/
    
4. Se define la región donde se va crear la infraestructura:

        REGION=us-east-1

5. Se importa el certificado del servicio Client VPN a AWS Certificate Manager (ACM):

        certserver=$(aws acm import-certificate --certificate fileb://~/certs/server.crt --private-key fileb://~/certs/server.key --certificate-chain fileb://~/certs/ca.crt --output text --region $REGION)

6. A continuación, se crea un grupo de logs en Amazon CloudWatch para registrar las conexiones de los clientes VPN:

        aws logs create-log-group --log-group-name client-vpn-log --region $REGION

8. Ahora se creará una infraestructura de VPC altamente disponible (en dos zonas de disponibilidad) para, posteriormente, definir dos interfaces de red para el punto de enlace de ClientVPN. Esta infraestructura dispondrá de dos subredes privadas, dos subredes públicas y dos Gatewa NAT. Para simplificar la tarea, se utilizará una plantilla de AWS CloudFormation (ubicada en `vpc/vpc.yaml`). Si no se tie

        aws cloudformation deploy --template-file vpc/vpc.yaml --stack client-vpn-stack --parameter-overrides file://vpc/client-vpn.json --region $REGION

9. Previamente a la definición del punto de enlace, se va a crear un AD mediante el servicio AWS Directory Service. AWS Directory Service permite crear:

   * AWS Managed Microsoft AD (AD completo)
   * Simple AD (AD sencillo, con las características más importantes)
   * AD Connector (proxy para un AD existente)
   
   En este caso, para simplificar el proceso y reducir el tiempo de despliegue, se creará un Simple AD. Para ello, es necesario obtener el Id de la VPC y los Ids de las subredes privadas donde residirá el AD

        vpcId=$(aws cloudformation describe-stacks --stack-name client-vpn-stack --query 'Stacks[].Outputs[?OutputKey==`VPC`].OutputValue' --output text --region $REGION)

        subnet1=$(aws cloudformation describe-stacks --stack-name client-vpn-stack --query 'Stacks[].Outputs[?OutputKey==`Privada1`].OutputValue' --output text --region $REGION)

        subnet2=$(aws cloudformation describe-stacks --stack-name client-vpn-stack --query 'Stacks[].Outputs[?OutputKey==`Privada2`].OutputValue' --output text --region $REGION)
        
   Por último, se crea el Simple AD utilizando como nombre de dominio `clientvpn.com` (se puede elegir el que se desee, es de ámbito local):

        ADId=$(aws ds create-directory --name clientvpn.com --password Client_VPN_AD$ --size Small --vpc-settings VpcId=$vpcId,SubnetIds=$subnet1,$subnet2 --output text --region $REGION)

10. A continuación, se añadirá un usuario de prueba al AD. Para ello, se crea un grupo de seguridad que permita el tráfico de entrada por el puerto 3389 TCP (RDP) y se lanza una instancia EC2 con sistema operativo Windows Server 2022, indicando su membresía en el AD anteriormente creado:

        rdpsg=$(aws ec2 create-security-group --group-name rdp-clientvpn-sg --description "Trafico entrada 3389 TCP" --vpc-id $vpcId --output text --query GroupId --region $REGION)

        aws ec2 authorize-security-group-ingress --group-id $rdpsg --protocol tcp --port 3389 --cidr 0.0.0.0/0 --region $REGION

        subredpublica=$(aws cloudformation describe-stacks --stack-name client-vpn-stack --query 'Stacks[].Outputs[?OutputKey==`Publica1`].OutputValue' --output text --region $REGION)

    Se crea un par de claves (en un entorno de <em>AWS Academy Learner Lab</em> se puede utilizar la clave `vockey`) y se lanza la instancia EC2:

        aws ec2 create-key-pair --key-name vpn-key --key-format pem --query KeyMaterial --output text --region $REGION > vpn-key.pem

        winVM=$(aws ec2 run-instances --image-id resolve:ssm:/aws/service/ami-windows-latest/Windows_Server-2022-Spanish-Full-Base --instance-type t3.large --security-group-ids $rdpsg --subnet-id $subredpublica --key-name vpn-key --query Instances[].InstanceId --output text --region $REGION)

    Hay que esperar a que la instancia esté en ejecución para obtener su DNS público:

        aws ec2 wait instance-running --instance-ids $winVM --region $REGION

    Se obtiene el nombre DNS público de la instancia

        nombredns=$(aws ec2 describe-instances --filters Name=instance-id,Values=$winVM --query 'Reservations[].Instances[][].PublicDnsName' --region $REGION --output text)

        echo $nombredns

    Se obtiene la contraseña del usuario `Administrador` mediante la orden:

        password=$(aws ec2 get-password-data --instance-id $winVM --priv-launch-key vpn-key.pem --region $REGION --query PasswordData --output text)

    Tras esperar unos 4-5 minutos, se puede obtener la contraseña:

        echo $password

11. Se lanza una conexión RDP utilizando el nombre DNS especificado en `$nombredns`, el usuario `Administrador` y la contraseña especificada en `$password`.

12. Se obtienen los dos servidores DNS del Simple AD creado:

        dns1=$(aws ds describe-directories --query "DirectoryDescriptions[?DirectoryId=='$ADId'].DnsIpAddrs[0]" --output text --region $REGION)

        dns2=$(aws ds describe-directories --query "DirectoryDescriptions[?DirectoryId=='$ADId'].DnsIpAddrs[1]" --output text --region $REGION)

13. Hay que unir la instancia EC2 al dominio `clientvpn.com`. Previamente se configuran los servidores DNS para poder resolver dicho dominio. Los servidores DNS de Simple AD resuelven el dominio `clientvpn.com` y redirigen el resto de consultas a los servidores administrados de AWS en la VPC. Se modifican los parámetros indicados en la configuración de la red:

![image-01.png](/images/image-01.png)

14. Para unir la instancia EC2 al dominio, se cambia del grupo de trabajo al dominio `clientvpn.com` y se introducen las credenciales del usuario `Administrator` del dominio con la contraseña indicada durante la creación del Simple AD:

![image-02.png](/images/image-02.png)

![image-03.png](/images/image-03.png)

Habrá que reiniciar la instancia EC2 tras el proceso. De nuevo, se realiza la conexión RDP con la instancia EC2, pero en esta ocasión hay que realizar la autenticación con el usuario Administrador del dominio, es decir `Administrator@clientvpn.com` y la contraseña indicada, en el caso de este ejemplo `Client_VPN_AD$`.

14. A continuación se instalan las `Herramientas de AD` para poder crear un usuario de AD de prueba, que se utilizará para la conexión al punto de enlace de Client VPN. Para ello se selecciona la opción 2 `Agregar roles y características` desde el `Administrador del Servidor`:

![image-04.png](/images/image-04.png)


15. En el asistente siguiente, en el apartado `Selección del servidor` se verifica que la instancia EC2 aparece en el Grupo de Servidores:

![image-05.png](/images/image-05.png)

16. Se marcan en el apartado `Características`, dentro del grupo `Herramientas de administración remota del servidor`, las opciones `Herramientas de AD DS y AD LDS` así como `Herramientas del servidor DNS` y se confirma la instalación:

![image-06.png](/images/image-06.png)

17. Por último, se accede a la herramienta `Usuarios y equipos de Active Directory` y se añade un usuario de prueba y se establece la contraseña:

![image-07.png](/images/image-07.png)

En la imagen anterior, se ha creado el usuario `joseemilio@clientvpn.com`.

18. A continuación, se define el punto de enlace de Client VPN en la VPC por defecto. El punto de enlace aceptará conexiones por el puerto 1194 UDP (OpenVPN). Se utilizará la red 10.8.0.0/24 para el túnel. El archivo `conf/authentication.json` contiene el esqueleto para indicar el AD que se utilizará para la autenticación que se va a personalizar con el Id del Simple AD creado. Para ello, desde la consola local se ejecuta:
        
        sed -i 's|<directorio>|'$ADId'|g' conf/authentication.json

        vpnId=$(aws ec2 create-client-vpn-endpoint --client-cidr-block 10.8.0.0/22 --server-certificate-arn $certserver --authentication-options file://conf/authentication.json --connection-log-options file://conf/log-options.json --transport-protocol udp --vpn-port 1194 --vpc-id $vpcId --query ClientVpnEndpointId --output text --region $REGION)

19. Ahora falta asociar las subredes donde residirán la interfaces de red del punto de enlace. A estas interfaces se le asignará (automáticamente) una IP privada dentro del rango de la subred en la que se encuentren. Es una buena práctica crear múltiples asociaciones en subredes (de preferencia privadas) en diferentes zonas de disponibilidad para disponer de múltiples interfaces para el punto de enlace de Client VPN y, por ende, tener un diseño resiliente:

        aws ec2 associate-client-vpn-target-network --client-vpn-endpoint $vpnId --subnet-id $subnet1 --region $REGION

        aws ec2 associate-client-vpn-target-network --client-vpn-endpoint $vpnId --subnet-id $subnet2 --region $REGION
        
    **Nota:** Las interfaces de red asociadas al punto de enlace de Client VPN pueden tener asignados grupos de seguridad (al fin y al cabo son ENIs). Sin embargo, con el objeto de simplificar este despliegue, se dejará asignado el grupo de seguridad `default` de la VPC creada. Esto permitirá que el cliente local pueda acceder a los recursos de la VPC que tengan asignado el grupo de seguridad `default` u otros grupos de seguridad que permitan en alguna de sus reglas de entrada el grupo de seguridad `default`.

20. En unos minutos, se habrán vinculado las interfaces de red en las subredes anteriores al punto de enlace de Client VPN. Sin embargo no se tendrá acceso a ninguna ubicación, ya que debe autorizarse explícitamente el acceso a los recursos accedidos a través de Client VPN. Es por ello que se necesita añadir autorizaciones a los CIDR necesarios; en este caso se va a permitir todo el direccionamiento a cualquier lugar (`0.0.0.0/0`):

        aws ec2 authorize-client-vpn-ingress --client-vpn-endpoint-id $vpnId --target-network-cidr 0.0.0.0/0 --authorize-all-groups --region $REGION 

21. Ahora sólo resta añadir las rutas estáticas a la tabla de rutas del punto de enlace de Client VPN. Para ello, vinculamos la ruta estática `0.0.0.0/0` en cada una de las subredes (privadas) donde se hayan definido interfaces de red sobre el punto de enlace de Client VPN:

        aws ec2 create-client-vpn-route --client-vpn-endpoint-id $vpnId --destination-cidr-block 0.0.0.0/0 --target-vpc-subnet-id $subnet1 --region $REGION

        aws ec2 create-client-vpn-route --client-vpn-endpoint-id $vpnId --destination-cidr-block 0.0.0.0/0 --target-vpc-subnet-id $subnet2 --region $REGION

22. Tras realizar los pasos anteriores, ya estaría configurado el punto de enlace de Client VPN. Ahora, desde la máquina cliente que vaya a conectar a la VPN se accede mediante un navegador web a la URL:

        https://self-service.clientvpn.amazonaws.com/

Esta URL es el portal de autoservicio de AWS Client VPN y va a permitir, introduciendo el Id del punto de enlace creado (almacenado en la variable `$vpnId`) y las credenciales del usuario del AD, poder obtener el archivo de configuración del cliente VPN. El nombre del Id de la Client VPN se puede obtener como:

        echo $vpnId

![image-08.png](/images/image-08.png)

![image-09.png](/images/image-09.png)

![image-10.png](/images/image-10.png)

Presionando el botón de `Download client configuration` se descargará el archivo de configuración.

23. Por último, se importa el perfil del archivo descargado con el cliente OpenVPN elegido. Realizar la conexión e introducir las credenciales del usuario del AD:

![image-11.png](/images/image-11.png)

16. Ejecutar el comando `route -n` para comprobar que la ruta por defecto tiene como puerta de enlace la IP del túnel creado por la conexión contra el punto de enlace de Client VPN.

17. Otra prueba que se puede realizar para la comprobación es lanzar una sesión RDP contra la IP privada de la instancia EC2 creada anteriormente.
