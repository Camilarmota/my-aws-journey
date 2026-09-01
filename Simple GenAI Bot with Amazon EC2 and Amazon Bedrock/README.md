# Simple GenAI Bot with Amazon EC2 and Amazon Bedrock

Este repositório contém as instruções e códigos para implantar uma aplicação web simples de Inteligência Artificial Generativa (um bot de resposta única ou *one-shot*) utilizando uma instância Amazon EC2 e o Amazon Bedrock.

A aplicação executa um script Python que invoca o modelo **Claude Sonnet 4.5** da Anthropic no Amazon Bedrock por meio de um Profile de Inferência.

---

## 🏗️ Arquitetura da Solução

O fluxo básico da aplicação funciona da seguinte forma:
1. O **Usuário** envia uma requisição de texto através de uma página web hospedada na instância EC2.
2. A **Instância EC2** executa o servidor web Apache (`httpd`) e processa a requisição usando um script Python em ambiente CGI.
3. A instância possui uma **Role do IAM** anexada com permissão `AmazonBedrockFullAccess`, permitindo que ela faça chamadas de API de forma segura.
4. O script Python faz uma chamada de API ao **Amazon Bedrock** utilizando o SDK `boto3` para solicitar a inferência do **Claude Sonnet 4.5** na região de Oregon (`us-west-2`).
5. A resposta processada é retornada e exibida na página web do usuário.

---

## 🛠️ Pré-requisitos

* Acesso ao Console AWS.
* Familiaridade geral com o Console AWS.
* **Região recomendada:** Oregon (`us-west-2`), onde os recursos do Bedrock necessários estão disponíveis por padrão.
* *Nota: Custos nominais de uso de EC2 e Bedrock serão aplicados.*

---

## 🚀 Passo a Passo de Implementação

### Passo 1: Inicializar a Instância EC2
1. No Console AWS, acesse o serviço **EC2** e clique em **Launch instance**.
2. Defina um nome para a instância.
3. Mantenha os padrões de AMI (**Amazon Linux 2023**) e tipo de instância (**t3.micro**).
4. Em **Key pair**, selecione **Proceed without a key pair** (não recomendado para produção, mas utilizaremos o *EC2 Instance Connect* para acesso simplificado neste laboratório).
5. Em **Network settings** (Configurações de rede):
   * Ative a opção **Allow SSH traffic from Anywhere (0.0.0.0/0)**.
   * Marque a caixa de seleção **Allow HTTP traffic from the internet**.
6. Clique em **Launch instance** e aguarde a inicialização.

### Passo 2: Criar e Associar a Role do IAM
A instância EC2 necessita de permissões para invocar o modelo no Amazon Bedrock.
1. No Console AWS, acesse o serviço **IAM** e clique em **Roles** > **Create role**.
2. Escolha **AWS service** como tipo de entidade confiável e selecione **EC2** na lista.
3. Na tela de políticas de permissão, busque por **`AmazonBedrockFullAccess`** e marque a caixa de seleção correspondente.
4. Avance, defina um nome apropriado para a Role e clique em **Create role**.
5. Associe a Role à instância EC2:
   * Retorne ao painel do **EC2**.
   * Selecione a instância criada, clique em **Actions** > **Security** > **Modify IAM role**.
   * Selecione a role criada anteriormente e clique em **Update IAM role**.

### Passo 3: Conectar e Instalar as Dependências no EC2
1. Selecione a instância e clique em **Connect** (Certifique-se de estar na aba **EC2 Instance Connect**).
2. Clique no botão laranja **Connect** para abrir o terminal web Linux.
3. Execute os seguintes comandos sequencialmente para atualizar o sistema e instalar o servidor web Apache e o Python:
   ```bash
   sudo yum update -y
   sudo yum install -y httpd
   sudo systemctl start httpd
   sudo systemctl enable httpd
   sudo yum install -y python3 python3-pip
   sudo pip3 install boto3
   ```

### Passo 4: Configurar a Interface Web (HTML)
Crie a página web que receberá as mensagens do usuário.
1. Crie o arquivo `index.html` no editor de texto:
   ```bash
   sudo vi /var/www/html/index.html
   ```
2. Pressione a tecla `i` para entrar no modo de inserção e cole o código HTML abaixo:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <title>Bedrock Text Processing</title>
   </head>
   <body>
       <h1>Enter Text for Processing</h1>
       <form action="/cgi-bin/process_text.py" method="post">
           <textarea name="user_input" rows="10" cols="50"></textarea><br>
           <input type="submit" value="Process Text">
       </form>
   </body>
   </html>
   ```
3. Pressione `Esc`, digite `:wq` e aperte `Enter` para salvar e fechar o arquivo.

### Passo 5: Configurar o Backend Python (CGI)
Crie o script em Python que se comunicará com o Amazon Bedrock.
1. Abra o arquivo para edição do script:
   ```bash
   sudo vi /var/www/cgi-bin/process_text.py
   ```
2. Pressione a tecla `i` e cole o seguinte código em Python:
   ```python
   #!/usr/bin/env python3
   import cgi
   import cgitb; cgitb.enable()
   import boto3
   import json

   print("Content-Type: text/html")
   print()

   form = cgi.FieldStorage()
   user_input = form.getvalue('user_input')

   if user_input:
       bedrock_client = boto3.client('bedrock-runtime', region_name='us-west-2')
       request_payload = {
           'anthropic_version': 'bedrock-2023-05-31',
           'max_tokens': 500,
           'temperature': 0.9,
           'top_k': 250,
           'messages': [
               {
                   'role': 'user',
                   'content': user_input
               }
           ]
       }
       response = bedrock_client.invoke_model(
           modelId='global.anthropic.claude-sonnet-4-5-20250929-v1:0',
           body=json.dumps(request_payload)
       )

       result = json.loads(response['body'].read())
       processed_text = "".join([output["text"] for output in result["content"]])

       print(f'<h2>Processed Text:</h2><p>{processed_text}</p><form action="/cgi-bin/process_text.py" method="post"><textarea name="user_input" rows="10" cols="50"></textarea><br><input type="submit" value="Process Text"></form>')
   else:
       print("<h2>No input provided.</h2>")
   ```
3. Pressione `Esc`, digite `:wq` e aperte `Enter` para salvar.
4. Torne o script executável e reinicie o servidor web rodando os comandos:
   ```bash
   sudo chmod +x /var/www/cgi-bin/process_text.py
   sudo systemctl restart httpd
   ```

---

## 🧪 Testando o Bot

1. No console do EC2, copie o **Public IPv4 address** (Endereço IP Público) da sua instância.
2. Abra uma nova aba no seu navegador web e cole o endereço IP.
3. A página simples com uma caixa de texto será exibida. Digite uma pergunta ou comando e clique em **Process Text** para receber a resposta gerada diretamente pelo Claude Sonnet 4.5 via Amazon Bedrock!

---

## 🧹 Limpeza dos Recursos

Para evitar cobranças indesejadas após concluir os testes:
1. Retorne ao console do **EC2**.
2. Selecione a instância, clique em **Instance state** > **Terminate instance** (Excluir).
