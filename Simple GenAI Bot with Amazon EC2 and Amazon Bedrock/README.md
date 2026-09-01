# 🚀 Simple GenAI Bot with Amazon EC2 and Amazon Bedrock

![Banner do Projeto](https://lh3.googleusercontent.com/notebooklm/AKYWMX8yUAJi5CAiwh_SWHkon7_MrO_npWTR5Ua53ZoLyetW7DvczSI_H5yVS5ARPWXxulySKl0U_fTxUdk5oWbnCWWVuLHwaUvQxsrJpYDthWVy-oI3XTF4lHn_hPoLrM5jdFmUfld5ckR-fUxrY04j3Z2Wxdmw8vY)

> 📌 **Nota sobre a Prévia no Gemini Notebook:** Se você estiver lendo este arquivo através do painel de visualização do Gemini Notebook, o banner acima deve carregar normalmente usando a URL dinâmica gerada pelo sistema. Para a imagem de arquitetura abaixo, você precisará visualizá-la quando o projeto for publicado no GitHub, utilizando o arquivo físico `imagem_2026-09-01_152042738.png` que você carregou como fonte.

Este repositório contém a documentação completa e o código-fonte para implementar um chatbot web de resposta única (*one-shot* chatbot) na nuvem da **Amazon Web Services (AWS)**. 

O projeto demonstra como configurar uma aplicação web ponta a ponta que recebe uma mensagem do usuário, autentica-se de forma segura usando permissões nativas do IAM e consome o modelo de linguagem avançado **Claude Sonnet 4.5** do **Amazon Bedrock**.

---

## 🏗️ Arquitetura do Sistema

O fluxo das requisições e a estrutura dos recursos provisionados seguem o diagrama abaixo:

![Arquitetura da Solução](imagem_2026-09-01_152042738.png)

### 🔄 Fluxo de Execução Técnica
1. **Requisição do Usuário:** O usuário digita uma pergunta na página web simples servida pelo Apache (`httpd`) rodando na porta 80.
2. **Servidor Web & CGI:** O servidor Apache recebe o payload via método `POST` e invoca o script de backend escrito em Python utilizando a interface CGI (*Common Gateway Interface*).
3. **Autenticação IAM Segura:** Em vez de usar credenciais estáticas no código (o que viola as práticas de segurança), a instância EC2 possui um **IAM Role** associado com a política de acesso `AmazonBedrockFullAccess` para se autenticar dinamicamente na AWS de forma segura.
4. **Chamada ao Amazon Bedrock:** O script Python utiliza o SDK **Boto3** para enviar o payload formatado ao serviço Amazon Bedrock na região `us-west-2` (Oregon), chamando o modelo **Claude Sonnet 4.5** via Inference Profile.
5. **Retorno da Resposta:** O Bedrock processa a entrada e devolve a resposta gerada, que é formatada dinamicamente pelo script CGI e renderizada no navegador do usuário.

---

## 🛠️ Tecnologias e Conceitos Utilizados

*   **Amazon EC2 (t3.micro & Amazon Linux 2023):** Servidor virtual que hospeda o frontend e executa o código de backend de forma econômica e eficiente.
*   **Amazon Bedrock (Claude Sonnet 4.5):** Serviço de IA generativa totalmente gerenciado pela AWS. A chamada utiliza parâmetros configurados como `temperature: 0.9` e `max_tokens: 500`.
*   **Boto3 SDK:** Biblioteca oficial da AWS para Python que encapsula e simplifica todas as chamadas de API. *Curiosidade técnica:* O nome "Boto" é uma homenagem direta ao boto-cor-de-rosa que vive no Rio Amazonas.
*   **Estatelessness (Falta de Estado):** O chatbot básico é um modelo *one-shot*, o que significa que o Claude não retém histórico ou contexto de mensagens anteriores nativamente. Cada envio de texto é uma transação isolada.

---

## 💻 Guia Prático de Implementação

### Passo 1: Provisionar a Instância EC2
1. Acesse o console da **AWS** na região de **Oregon (us-west-2)**.
2. Vá até o serviço **EC2** e clique em **Launch instance**.
3. Nomeie sua máquina e selecione os seguintes parâmetros padrões:
   *   **AMI:** Amazon Linux 2023.
   *   **Instance type:** `t3.micro` (elegível para nível gratuito).
   *   **Key pair:** Selecione a opção **Proceed without a key pair** (acessaremos diretamente via console através do *EC2 Instance Connect*).
4. Em **Network Settings**:\
   *   Mantenha a opção **Allow SSH traffic from Anywhere (0.0.0.0/0)** ativa.
   *   Marque a caixa de seleção **Allow HTTP traffic from the internet** para liberar o tráfego web público na porta 80.
5. Clique em **Launch instance**.

### Passo 2: Configurar a Identidade do IAM (Segurança Nativa)
Para que a instância EC2 possa se comunicar com o Bedrock sem expor chaves de acesso vulneráveis no código:
1. No Console AWS, navegue até o serviço **IAM** (Identity and Access Management).
2. No menu lateral esquerdo, clique em **Roles** e depois no botão **Create role**.
3. Em **Trusted entity type**, selecione **AWS service**. No menu de seleção rápida abaixo, selecione **EC2** e clique em **Next**.
4. Na barra de pesquisa de políticas, procure por **`AmazonBedrockFullAccess`**. Marque a caixa de seleção ao lado da política e clique em **Next**.
5. Dê um nome significativo para a sua role (ex: `restart-bot-role`) e clique em **Create role** para concluir.
6. Associe a Role à sua instância EC2:
   *   Volte para o console do **EC2** e clique em **Instances**.
   *   Marque a caixa da sua instância ativa.
   *   Clique no botão superior de menu **Actions** > **Security** > **Modify IAM role**.
   *   Selecione a role criada no menu suspenso e clique em **Update IAM role**.

### Passo 3: Configurar o Servidor e Instalar Dependências
1. No painel de instâncias do EC2, selecione a sua máquina e clique no botão azul **Connect** no topo da tela.
2. Certifique-se de estar na aba **EC2 Instance Connect** e clique no botão laranja **Connect** no canto inferior direito. Uma nova aba com o terminal Linux se abrirá no seu navegador.
3. Execute os seguintes comandos sequencialmente para atualizar o sistema operacional, instalar o servidor web Apache e configurar o ambiente Python:
   ```bash
   # Atualizar pacotes do sistema
   sudo yum update -y

   # Instalar, iniciar e habilitar o servidor Apache HTTPD
   sudo yum install -y httpd
   sudo systemctl start httpd
   sudo systemctl enable httpd

   # Instalar o Python 3, o gerenciador de pacotes pip e o SDK boto3
   sudo yum install -y python3 python3-pip
   sudo pip3 install boto3
   ```

### Passo 4: Implementar o Frontend (HTML)
1. Crie o arquivo de entrada do site utilizando o editor de texto `vi` direto no terminal da sua instância:
   ```bash
   sudo vi /var/www/html/index.html
   ```
2. No teclado, aperte a tecla `i` para ativar o modo de inserção de texto do editor.
3. Cole o seguinte código estruturado HTML:
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
4. Pressione a tecla `Esc` para sair do modo de inserção, digite `:wq` (escrever e sair) e aperte `Enter` para salvar o arquivo.

### Passo 5: Implementar o Backend CGI (Python)
1. Abra um novo arquivo para o script Python no diretório CGI do Apache utilizando o editor `vi`:
   ```bash
   sudo vi /var/www/cgi-bin/process_text.py
   ```
2. Pressione a tecla `i` para entrar no modo de inserção e cole o código abaixo:
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
       # Cria o cliente do Bedrock Runtime na região Oregon (us-west-2)
       bedrock_client = boto3.client('bedrock-runtime', region_name='us-west-2')
       
       # Payload de requisição formatado conforme o padrão do Claude Sonnet 4.5
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
       
       # Invoca o modelo Claude Sonnet 4.5 via Bedrock
       response = bedrock_client.invoke_model(
           modelId='global.anthropic.claude-sonnet-4-5-20250929-v1:0',
           body=json.dumps(request_payload)
       )

       # Decodifica e extrai a resposta textual do JSON retornado
       result = json.loads(response['body'].read())
       processed_text = "".join([output["text"] for output in result["content"]])

       # Renderiza a resposta dinâmica na página mantendo a caixa de texto ativa
       print(f'<h2>Processed Text:</h2><p>{processed_text}</p><form action="/cgi-bin/process_text.py" method="post"><textarea name="user_input" rows="10" cols="50"></textarea><br><input type="submit" value="Process Text"></form>')
   else:
       print("<h2>No input provided.</h2>")
   ```
3. Aperte `Esc`, digite `:wq` e pressione `Enter` para salvar o script Python.
4. Aplique permissão de execução ao script Python e garanta que o Apache consiga ler as modificações:
   ```bash
   sudo chmod +x /var/www/cgi-bin/process_text.py
   sudo systemctl restart httpd
   ```

---

## 🧪 Validando a Solução (Passo a Passo)

1. Vá ao console do **EC2** na AWS e copie o **Public IPv4 address** (IP público) da sua máquina.
2. Abra uma nova aba no seu navegador web e digite o endereço IP público (Ex: `http://54.212.x.x`).
3. Digite sua pergunta na caixa de texto do bot e clique em **Process Text**. 
4. O modelo de linguagem do **Claude Sonnet 4.5** responderá diretamente na tela em alguns segundos!

---

## 🧹 Boas Práticas e Desativação (Cleanup)

Para evitar surpresas na sua fatura da AWS após testar sua aplicação, lembre-se sempre de excluir os recursos criados:
1. Retorne ao console do **EC2** no painel da AWS.
2. Selecione a instância criada para este laboratório.
3. No topo, clique em **Instance state** > **Terminate instance** para desligar e remover a máquina virtual permanentemente.
