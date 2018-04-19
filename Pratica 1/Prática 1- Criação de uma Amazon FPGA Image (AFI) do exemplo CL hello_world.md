DETI/UFC - Cursos de Eng. de Computação e Eng. de Telecomunicações

Elaborada por Jardel Silveira e Vanessa Rodrigues 

# **Criação de uma Amazon FPGA Image (AFI) do exemplo CL hello_world**

**Descrição**

Nesta prática vamos conectar e configurar a instância t2.2xlarge do EC2 para a implementação e sintetização de um exemplo disponível no AWS EC2 FPGA Hardware and Software Development Kits. Além disso, vamos conectar e configurar a instância  f1.2xlarge para carregar o projeto sintetizado e testá-lo. 

O exemplo utilizado será o cl_hello_world, um exemplo simples que demonstra a conectividade básica Shell-para-Cl, instâncias de registradores com mapeamento de memória e o uso dos switches Virtual LED e DIP. Nesse exemplo são implementados dois registradores no Espaço de memória FPGA AppPF BAR0 ([FPGA PCIe memory space overview](https://github.com/aws/aws-fpga/blob/master/hdk/docs/AWS_Fpga_Pcie_Memory_Map.md)) conectado à interface OCL AXI-L. Os registradores são os seguintes:

1. Hello World  (offset 0x500)

2. Virtual LED  (offset 0x504)

O Hello World  é um registrador de leitura/escrita de 32 bits. Para demonstrar o acesso correto a esse registrador, os dados escritos no registrador serão reorganizados (byte swapp), tornando os bits mais significativos como menos significativos e vice-versa. No exemplo, o valor escrito no registrador será 0xefbeadde e o valor lido após o swapp será 0xdeadbeef.

O Virtual LED é um registrador de somente leitura de 16 bits, que "sombreia" os 16 bits menos significativos do registrador Hello World, de modo que ele mantenha o mesmo valor dos bits 15: 0 que foram escritos no registrador Hello World. 

O design do exemplo hello_world utiliza o Virtual LED e um DIP switch que consistem em dois sinais descritos no arquivo (./../../../common/shell_stable/design/interfaces/cl_ports.vh). 

 ```bash
 input[15:0] sh_cl_status_vdip,               //Virtual DIP. 
   
 output logic[15:0] cl_sh_status_vled,        //Virtual LEDs 
 ```

Neste exemplo, o registrador Virtual LED é usado para direcionar o sinal do LED virtual,  cl_sh_status_vled, e o Virtual DIP switch, sh_cl_status_vdip, é usado para "gatilhar" o valor do registrador Virtual LED enviado ao LED virtual. Por exemplo, se o sh_cl_status_vdip é setado para 16'h00FF, então apenas os 8 bits do registrador Virtual LED serão sinalizados no sinal do LED virtual cl_sh_status_vled. Porém, se o sh_cl_status_vdip é setado para 16'hFFFF, então os 16 bits do registrador Virtual LED serão sinalizados no sinal do LED virtual cl_sh_status_vled.

**Objetivos de Aprendizagem**

*  Conexão e configuração das instâncias t2.2xlarge e f1.2xlarge.

* Sintetização do projeto hello_world (geração do arquivo .tar).

* Download da AFI, gerada a partir do projeto hello_world, na FPGA da instância f1.2xlarge.

* Execução do teste do projeto hello_world.

**Parte 1 - Configurar o Amazon EC2**
 

Antes de iniciar uma instância EC2 é necessário fazer algumas configurações. Para esta prática serão necessárias apenas as configurações de  criar um security group e uma key-pair. Essas configurações serão descritas nos ítens a seguir.


   1. Acesse o console da conta pelo link https://console.aws.amazon.com/ec2/v2/home. No canto superior direito da tela, escolha a região **Us East (N. Vírginia)**. 
       
   2. No menu, em  **NETWORK E SECURITY**  selecione a opção **Security Group**. Clique em **Create Security Group**.

   3. Na tela seguinte, adicione o nome do Security Group. Em seguida, adicione a descrição. Na guia **Inbound**, crie as regras mostradas na imagem abaixo. Escolha **Add Rule** para cada nova regra e, em seguida, clique em **Create**. 
        
         ![image alt text](image_4.jpeg)
     
     
   4. No menu, em  **NETWORK E SECURITY**  selecione a opção **Key Pairs**. Clique em **Create Key Pair**.
      
   5.  Atribua um nome para a Key Pair. Em seguida, clique em  **Create**.
     
   6. Após isso, será realizado o download do arquivo .pem, que deverá ser guardado em um diretório de fácil acesso.




**Parte 2 - Iniciar e conectar à instância**

1. Criar e conectar uma instância t2.2xlarge com o Ambiente de desenvolvimento **FPGA Developer AMI.**

    1. Acesse https://console.aws.amazon.com .  Você deve estar logado em sua conta Amazon.
    2. No canto superior direito, escolha a região **Us East (N. Vírginia)**.
    3. Clique em **EC2**.
    4. No menu do canto esquerdo, clique em **instances**.
    5. Na tela de Instâncias, clique em **Launch Instance**.
    6. No primeiro passo, você deve escolher a **AMI da instância**. Clique em **Aws Marketplace**.
    7. Na barra de procura, digite **FPGA Developer AMI** e clique em **select** e depois clique em **continue**. 
    8. Em **Choose Instance Type**, escolha **t2.2xlarge**.
    9. Na aba **Configure Security Group**, clique em **Select an existing security group** e escolha aquele criado na parte 1.
    10. Finalmente, clique em **Review and Launch** e depois clique em **Launch**.
    11. Em **Choose an Existing Key Pair**, escolha aquela criada na parte 1, selecione o checkbox para aceitar as permissões e clique em **Launch Instances**.
    12. Para verificar o estado da instância clique em **View Instances**.
    13. Verifique se a instância passou suas verificações de status, ou seja, a informação do status é **2/2 checks pass**. Você pode visualizar essas informações na coluna Status Checks na página Instances.
    14. Para conectar a instância, no caso de usuários linux, selecione a instância e clique em conectar, siga os passos informando na janela que será aberta. Obs: No comando informado, substitua “root” por “centos”.
    15. Para conectar a instância, no caso de usuários do Windows, deve-se seguir esse tutorial http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html.


**Parte 3 - Criar e Registrar uma Amazon FPGA Image (AFI)**

1. Configure o HDK e configure o CLI AWS

```bash 
$ git clone https://github.com/aws/aws-fpga.git $AWS_FPGA_REPO_DIR
$ cd $AWS_FPGA_REPO_DIR
$ source hdk_setup.sh
```

Obs: Ao usar a FPGA developer AMI a variável AWS_FPGA_REPO_DIR corresponde ao diretório /home/centos/src/project_data/aws-fpga.

Configure o AWS CLI (`aws configure`) inserindo as mesmas informações usadas na parte 1.

OBS: suas credenciais podem ser encontradas na página [https://console.aws.amazon.com/iam/home?#/security_credential](https://console.aws.amazon.com/iam/home?#/security_credential) 

2. Mudando para o diretório do exemplo **cl_hello_world** 

```bash
$ cd $HDK_DIR/cl/examples/cl_hello_world
$ export CL_DIR=$(pwd)
```

3. Construindo a Custom Logic (CL)

Nesta etapa será gerado um DCP, que é um arquivo .tar, para criar a Custom Logic. A geração do DCP pode demorar até 3 horas para completar, porém é possível ser notificado via-email quando a compilação for concluída. Para isso, é necessário configurar notificação via SN:

```bash
$ export EMAIL=your.email@example.com
$ $HDK_COMMON_DIR/scripts/notify_via_sns.py
```

Após isso, é necessário verificar o e-mail e confirmar a assinatura. Uma vez que a compilação esteja completa, um e-mail será enviado notificando que a compilação foi concluída, ou seja, o DCP foi gerado.

O formato do arquivo gerado será YY_MM_DD-hhmm.Developer_CL.tar e após ser gerado estará disponível no diretório  $CL_DIR/build/checkpoints/to_aws/. Caso a configuração notificação via SN não tenha sido realizada, é necessário ficar verificando neste diretório se o arquivo já está disponível.

Para gerar o DCP use os seguintes comandos:

```bash
$ vivado -mode batch   # Verificar se o vivado está instalado    
$ cd $CL_DIR/build/scripts
$ ./aws_build_dcp_from_cl.sh -notify  #Executar o script para converter o CL design para DCP. 
```
	
4. Submetendo o Design Checkpoint para a AWS criar a AFI

Após o arquivo .tar ser gerado, é necessário que seja criado um bucket no S3 e seja feito o upload do arquivo tarball no bucket. 

Para fazer o upload do arquivo tarball para S3, podem ser usadas qualquer uma das [ferramentas suportadas pelo S3.](https://docs.aws.amazon.com/AmazonS3/latest/dev/UploadingObjects.html) Por exemplo, você pode usar a CLI AWS da seguinte maneira:

```bash
$ aws s3 mb s3://<bucket-name> --region <region>   # Criar um  bucket no S3 (Escolha um nome único para o bucket)
$ aws s3 mb s3://<bucket-name>/<dcp-folder-name>/   # Criar uma pasta para seu arquivo tarball 

$ aws s3 cp $CL_DIR/build/checkpoints/to_aws/*.Developer_CL.tar \ s3://<bucket-name>/<dcp-folder-name>/ # Fazer o upload do arquivo para o S3        

$ aws s3 mb s3://<bucket-name>/<logs-folder-name>/  # Criar uma pasta para guardar seu arquivo de log
$ touch LOGS_FILES_GO_HERE.txt                     # Criar um arquivo temporário (temp file)
$ aws s3 cp LOGS_FILES_GO_HERE.txt s3://<bucket-name>/<logs-folder-name>/ #Copiar o arquivo de log para a pasta criada
```
   

Para criar a AFI use o seguinte comando:

```bash
$ aws ec2 create-fpga-image \
        --region <region> \
        --name <afi-name> \
        --description <afi-description> \
        --input-storage-location Bucket=<dcp-bucket-name>,Key=<path-to-tarball> \
        --logs-storage-location Bucket=<logs-bucket-name>,Key=<path-to-logs> \
	[ --client-token <value> ] 
```


A saída desse comando é composta dois identificadores referentes a AFI criada:

* FPGA Image Identifier ou AFI ID: este é o ID principal, usado para gerenciar a AFI através dos comandos AWS EC2 CLI e AWS SDK APIs. Este ID é regional, ou seja, se um AFI é copiado em várias regiões, ele terá uma AFI ID única diferente em cada região. Um exemplo de AFI ID é afi-06d0ffc989feeea2a.

* Global FPGA Image Identifier ou AGFI ID: esta é uma identificação global que é usada para se referir a um AFI dentro de uma instância F1. Por exemplo, para carregar ou limpar um AFI de um slot FPGA, você usa o AGFI ID. Uma vez que as  AGFI IDs são globais (por design), permite copiar uma combinação de AFI / AMI para várias regiões, e elas funcionarão sem requerer nenhuma configuração adicional. Um exemplo AGFI ID é agfi-0f0e045f919413242.

O comando  de descrição-fpga-images permite verificar o estado da AFI durante o processo de geração. É preciso fornecer o FPGA Image Identifier retornado, substitua no comando abaixo:

```bash
$ aws ec2 describe-fpga-images --fpga-image-ids afi-016fd6ccf3c73bf28
```

A AFI só pode ser carregada em uma instância F1 após a conclusão da sua geração e o estado AFI está configurado para disponível, como no seguinte exemplo:
```
{
        "FpgaImages": {
            {
			   
                "State": {
                    "Code": "available"
                },
			    ...
                "FpgaImageId": "afi-06d0ffc989feeea2a",
			    ...
            }
        ]
    }
```
Após a conclusão da geração da AFI, a AWS colocará os logs na localização do bucket (s3: // <nome do bucket> / <logs-pasta-name>) fornecido pelo desenvolvedor. A presença desses logs é uma indicação de que o processo de criação está completo.



**Parte 3 - Carregar e testar uma AFI registrada em uma instância F1**

Para realizar os próximos passos, será necessário iniciar uma instância F1. Para isso, siga os procedimentos da  Parte 2 e  substitua o parâmetro do tipo de instância para ```--instance-type f1.2xlarge```.

5. Configuração de ferramentas de gerenciamento AWS FPGA

Faça o download das ferramentas de gerenciamento FPGA, que são necessárias para carregar uma AFI em um FPGA, e configure o ambiente. Utilize os comandos abaixo:

```bash
$ git clone https://github.com/aws/aws-fpga.git $AWS_FPGA_REPO_DIR
$ cd $AWS_FPGA_REPO_DIR
$ source sdk_setup.sh
```
 	

Configure as credenciais do AWS Cli como no item 2 da parte 1.
```bash
$ aws configure         # Setar suas credenciais 
```

OBS: suas credenciais podem ser encontradas na página [https://console.aws.amazon.com/iam/home?#/security_credential](https://console.aws.amazon.com/iam/home?#/security_credential).

 

6. Carregar a AFI

Para certificar que qualquer AFI que tenha sido carregada anteriormente no slot esteja limpa, é necessário usar o seguinte comando:

```bash
$ sudo fpga-clear-local-image  -S 0
```

Para verificar se o espaço está limpo, é necessário usar o comando:

 ```bash
  $ sudo fpga-describe-local-image -S 0 -H
 ```

Se o espaço estiver limpo, a saída do comando será a seguinte:

![image alt text](image_1.png)

Se a descrição retorna um status 'Ocupado', a FPGA ainda está executando a operação anterior em segundo plano. É necessário aguardar até que o status seja 'cleared' como acima.

Para carregar a AFI na FPGA é necessário usar o comando abaixo, substituindo o AGFI ID da AFI criada.
```bash 
 $ sudo fpga-load-local-image -S 0 -I agfi-09ed851c9ba0e59f0
```

Após isso é necessário  verificar se a AFI foi carregado corretamente. A saída mostra o FPGA no estado "loaded" após a operação "load" da imagem FPGA, como abaixo:

```bash
   $ sudo fpga-describe-local-image -S 0 -R -H
```

  ![image alt text](image_2.png)

7. Validando usando o Software de Exemplo CL

	Cada CL exemplo vem com um software de tempo de execução sob $ CL_DIR / software / runtime / subdiretório. é necessário "construir" o aplicativo de tempo de execução que corresponda ao AFI carregado, da seguinte forma:

```bash
$ cd $HDK_DIR/cl/examples/cl_hello_world    
$ export CL_DIR=$(pwd)
$ cd $CL_DIR/software/runtime/
$ make all
$ sudo ./test_hello_world
```

A saída será a seguinte:

![image alt text](image_3.png)

**Parte 4: Fechando a Sessão**

Depois de terminar sua sessão, você pode "Parar" ou "Terminar" sua instância. Se você 'Terminar' a instância, seu volume raiz será excluído. Você precisará criar e configurar uma nova instância na próxima vez que precisar trabalhar na F1. Se você parar a instância, o volume do root é preservado e a instância interrompida pode ser reiniciada mais tarde, não precisando mais passar por etapas de configuração. A AWS não cobra por instâncias interrompidas, mas pode cobrar por qualquer volume EBS anexado à instância.

* Feche a sessão remota 
```bash
$ exit
```

* Retorne para o EC2 Dashboard: [https://console.aws.amazon.com/ec2](https://console.aws.amazon.com/ec2)

* Selecione **Instances** no menu lateral esquerdo.

* Selecione a Instância que está sendo executada, clique **Actions**, escolha **Instance State** e em seguida, clique em **Terminate**.

* Selecione **Elastic Block Store** no menu lateral esquerdo e clique em **Volumes**.

* Selecione os volumes listados na tela, clique em **Actions**, e em seguida, clique em **Delete Volumes**.

	

	

**Referências**

* Amazon Web Services. Hardware Development Kit (HDK) e Software Development Kit (SDK) [internet]. [Acesso em: 26 dez. 2017]. Disponível em: https://github.com/aws/aws-fpga/tree/master/hdk/cl/examples

* Amazon Web Services. Instâncias F1 do Amazon EC2 [internet]. [Acesso em: 26 dez. 2017]. Disponível em: [https://aws.amazon.com/pt/ec2/instance-types/f1/](https://aws.amazon.com/pt/ec2/instance-types/f1/)

* Amazon Web Services. Documentação do Amazon Elastic Compute Cloud [internet]. [Acesso em: 26 dez. 2017]. Disponível em: https://aws.amazon.com/pt/documentation/ec2/


