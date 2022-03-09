﻿Iniciando a Integração
================================

Solicitação de Configuração
++++++++++++++++++++++++++++++++

É premissa de toda aplicação, que irá consumir os serviços da API de assinatura avançada, estar integrada a Plataforma de Autenticação Digital do Cidadão -  `Login Único`_. Ainda assim, a autorização de acesso utilizada pela assinatura é condicionada ao processo de autorização explícita do usuário, conforme `Lei n° 14.063`_ Art.4º. O usuário deve conceder a autorização para Assinatura API Service assinar digitalmente um documento em nome deste usuário e essa autorização é solicitada durante o fluxo de autorização OAuth da API de assinatura. Por esse motivo que a liberação de acesso para emissão do certificado implica a geração de uma requisição ao servidor OAuth que controla os recursos desta API. 

Para consumir os serviços da API de assinatura digital gov.br, há necessidade da liberação de credencial do ambiente de homologação. Esta liberação ocorre por meio de envio das informações listadas abaixo: 

1. **URL de retorno para cadastramento da aplicação cliente**
2. **Chave PGP** - A chave PGP é solicitada para envio das credenciais de autenticação de forma segura, isto é, criptografada. Informações sobre como gerar chaves PGP e envio da chave pública, podem ser verificadas no último tópico deste roteiro.
3. **Endereço de e-mail do destinatário** para recebimento das credenciais; 
4. **Volumetria anual estimada da quantidade de documentos que serão assinados**. 
5. **Sazonalidade de uso da aplicação cliente. Informar o período de aumento da demanda, caso ocorrer**.
6. **Estimativa da quantidade de usuários únicos da aplicação cliente**.

Essas informações deverão ser encaminhadas, para o e-mail **int-assinatura-govbr@economia.gov.br** da Secretaria de Governança Digital (SGD) do Ministério da Economia (ME), por e-mail de um representante legal do órgão ou entidade responsável pelo serviço a ser integrado. A liberação do ambiente de produção ocorrerá somente após a homologação final validada com os integrantes da SGD/ME. 

Orientações para testes em ambiente de homologação 
+++++++++++++++++++++++++++++++++++++++++++++++++++

De Acordo com a portaria `SEDGGME Nº 2.154/2021`_ as identidades digitais da plataforma gov.br são classificadas em três tipos: Bronze, Prata e Ouro. A identidade bronze permite ao usuário somente a realização de assinaturas simples. Nesta plataforma para realizar uma assinatura avançada, seja qual for o ambiente, o usuário deve possuir identidade digital prata ou ouro. Caso o usuário não possua este nível de identidade, a aplicação cliente deverá emitir mensagem informando ao usuário. Segue um exemplo de mensagem:                             
"Prezado solicitante, para realizar a(s) assinatura(s) é necessário que a sua identidade na plataforma gov.br seja "Prata" ou "Ouro". Para a obtenção das identidades digitais requeridas por meio do qual o cidadão terá acesso ao serviço de assinatura e a outros serviços, a aplicação cliente deve direcionar o usuário ao serviço de Catálogo de Confiabilidades. Os parâmetros para requisição deste serviço estão descritos no roteiro de integração do Login Único no link https://manual-roteiro-integracao-login-unico.servicos.gov.br/pt/stable/iniciarintegracao.html#acesso-ao-servico-de-catalogo-de-confiabilidades-selos

Para realizar testes, no ambiente de homologação, o testador deve criar uma conta seguindo os passos deste `Tutorial conta ID prata <https://github.com/servicosgovbr/manual-integracao-assinatura-eletronica/raw/main/arquivos/Tutorial%20conta%20prata.pdf>`_. 

.. note:: No ambiente de testes é possível criar conta teste para qualquer CPF. 

.. attention:: 
  Somente os documentos assinados em ambiente de **PRODUÇÃO** podem ser validados no Verificador de Conformidade do ITI https://verificador.iti.br/.
  Documentos assinados digitalmente em ambiente de **HOMOLOGAÇÃO** podem ser verificados em: https://verificador.staging.iti.br/. 

API de assinatura digital gov.br
++++++++++++++++++++++++++++++++

A partir de agora, será feita uma revisão sobre a arquitetura de serviço, alguns conceitos utilizados pela plataforma e os detalhes da estrutura da API REST para assinatura digital utilizando certificados avançados gov.br.

A API adota o uso do protocolo OAuth 2.0 para autorização de acesso e o protocolo HTTP para acesso aos endpoints. Deste modo, o uso da API envolve duas etapas:

1. Geração do token de acesso OAuth (Access Token)

2. Acesso ao serviço de assinatura

Geração do Access Token
+++++++++++++++++++++++

Para geração do Access Token é necessário redirecionar o navegador do usuário para o endereço de autorização do servidor OAuth, a fim de obter seu consentimento para o uso de seu certificado para assinatura. Nesse processo, a aplicação deve usar credenciais previamente autorizadas no servidor OAuth. As seguintes credencias podem ser usadas para testes:

==================  ======================================================================
**Paramêtros**  	**Valor**
------------------  ----------------------------------------------------------------------
**Servidor OAuth**  https://cas.staging.iti.br/oauth2.0
**client_id**       devLocal
**secret**          younIrtyij3
**redirect_uri**    http://127.0.0.1:x/xx
==================  ======================================================================

As credenciais para o client_id “devLocal” estão configuradas no servidor OAuth para aceitar qualquer aplicação executando localmente (host 127.0.0.1, qualquer porta, qualquer caminho). Aplicações remotas não poderão usar essas credenciais de teste.

A URL usada para redirecionar o usuário para o formulário de autorização, conforme a especificação do OAuth 2.0, é a seguinte:

.. code-block:: console

	https://<Servidor OAuth>/authorize?response_type=code&redirect_uri=<URI de redirecionamento>&scope=sign&client_id=<client_id>

Neste endereço, o servidor OAuth faz a autenticação e pede a autorização expressa do usuário para acessar seu certificado para assinatura. Neste instante será pedido um código de autorização a ser enviado por SMS. 

.. important::
  **EM HOMOLOGAÇÃO**, NÃO SERÁ ENVIADO SMS, DEVE-SE USAR O CÓDIGO **12345**.

Após a autorização, o servidor OAuth redireciona o usuário para o endereço <URI de redirecionamento> especificado e passa, como um parâmetro de query, o atributo Code. O <URI de redirecionamento> deve ser um endpoint da aplicação correspondente ao padrão autorizado no servidor OAuth, e capaz de receber e tratar o parâmetro “code”. Este atributo deve ser usado na fase seguinte do protocolo OAuth, pela aplicação, para pedir um Access Token ao servidor OAuth, com a seguinte requisição HTTP com método POST para o endereço https://cas.staging.iti.br/oauth2.0/token? passando as seguintes informações:

==================  ======================================================================
**Paramêtros**  	**Valor**
------------------  ----------------------------------------------------------------------
**code**            Código de autenticação gerado pelo provedor. Será utilizado para obtenção do Token de Resposta. Possui tempo de expiração e só pode ser utilizado uma única vez.
**client_id**       devLocal
**grant_type**      authorization_code
**client_secret**	younIrtyij3
**redirect_uri**    http://127.0.0.1:x/xx
==================  ======================================================================

.. code-block:: console

	https://cas.staging.iti.br/oauth2.0/token?code=<code>&client_id=<clientId>&grant_type=authorization_code&client_secret=<secret>&redirect_uri=<URI de redirecionamento>

O <URI de redirecionamento> deve ser exatamente o mesmo valor passado na requisição “authorize” anterior. O servidor OAuth retornará um objeto JSON contendo o Access Token, que deve ser usado nas requisições subsequentes aos endpoints do serviço.

.. important::
  O servidor OAuth de homologação está delegando a autenticação ao ambiente de **staging** do gov.br

.. important::
  O access token gerado autoriza o uso da chave privada do usuário para a confecção de **uma** única assinatura eletrônica avançada. O token deve ser usado em até 10 minutos. O tempo de validade do token poderá ser modificado no futuro à discrição do ITI.

Obtenção do certificado do usuário
++++++++++++++++++++++++++++++++++

Para obtenção do certificado do usuário deve-se fazer uma requisição HTTP GET para endereço https://assinatura-api.staging.iti.br/externo/v2/certificadoPublico enviando o cabeçalho Authorization com o tipo de autorização Bearer e o access token obtido anteriormente. Segue abaixo o parâmetro do Header para requisição:

==================  ======================================================================
**Paramêtros**  	**Valor**
------------------  ----------------------------------------------------------------------
**Authorization**   Bearer <access token>
==================  ======================================================================

Exemplo de requisição:

.. code-block:: console

		GET /externo/v2/certificadoPublico HTTP/1.1
		Host: assinatura-api.staging.iti.br 
		Authorization: Bearer AT-183-eRE7ot2y3FpEOTCIo1gwnZ81LMmT5I8c

Será retornado o certificado digital em formato PEM na resposta.


Realização da assinatura digital de um HASH SHA-256 em PKCS#7
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Para gerar um pacote PKCS#7 contendo a assinatura digital de um HASH SHA-256 utilizando a chave privada do usuário, deve-se fazer uma requisição HTTP POST para o endereço https://assinatura-api.staging.iti.br/externo/v2/assinarPKCS7 enviando os seguintes paramêtros:

==================  ======================================================================
**Paramêtros**  	**Valor**
------------------  ----------------------------------------------------------------------
**Content-Type**    application/json       
**Authorization**   Bearer <access token>
==================  ======================================================================

Body da requisição:

.. code-block:: JSON

	{ "hashBase64": "<Hash SHA256 codificado em Base64>"} 

Exemplo de requisição:

.. code-block:: console

		POST /externo/v2/assinarPKCS7 HTTP/1.1
		Host: assinatura-api.staging.iti.br 
		Content-Type: application/json	
		Authorization: Bearer AT-183-eRE7ot2y3FpEOTCIo1gwnZ81LMmT5I8c

		{"hashBase64":"kmm8XNQNIzSHTKAC2W0G2fFbxGy24kniLuUAZjZbFb0="}

Será retornado um arquivo contendo o pacote PKCS#7 com a assinatura digital do hash SHA256-RSA e com o certificado público do usuário. O arquivo retornado pode ser validado em https://verificador.staging.iti.br/.

API de Verificação de Conformidade do Padrão de Assinaturas Digitais
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Os serviços de verificação de Conformidade do Padrão de Assinatura Digital objetivam aferir a conformidade de assinaturas digitais existentes em um arquivo assinado.Se destinam à comunidade e organizações públicas e privadas que desenvolvem aplicativos geradores de assinatura digital para auxiliar na verificação da conformidade de arquivos assinados, resultantes de seus códigos, em conformidade com as especificações. 
Esta API contém dois serviços que utilizam o cabeçalho Content-Type: multipart/form-data, conforme especificado na tabela abaixo:

==================  ======================================================================
**Cabeçalho**       **Valor**
------------------  ----------------------------------------------------------------------
**Content-Type**    multipart/form-data       
==================  ======================================================================

* Requisição POST https://verificador.staging.iti.br/inicio 

Realiza uma análise preliminar sobre os artefatos de assinatura digital identificando se o arquivo contém pelo menos uma assinatura e se a assinatura é destacada. Body da requisição especificados na tabela abaixo:

=====================  ======================================================================
**Request body**       **Valor**
---------------------  ----------------------------------------------------------------------
**signature_files[]**  Array de arquivos de assinatura  
**detached_files[]**   Array de arquivos assinados - Somente para assinatura detached!
=====================  ======================================================================
         
Exemplo de requisição:

.. code-block:: console

		POST 'https://verificador.staging.iti.br/inicio' \
		--header 'Content-Type: multipart/form-data' \
		--form 'signature_files[]=@"/path/to/file/response.p7s"' \
		--form 'detached_files[]=""'

* Requisição POST https://verificador.staging.iti.br/report 

Realiza a verificação de assinaturas digitais em arquivos retornando o relatório de verificação de assinaturas no formato desejado. Body da requisição especificados na tabela abaixo:

=====================  ======================================================================
**Request body**       **Valor**
---------------------  ----------------------------------------------------------------------
**report_type**        Formato desejado do relatório de saída (json/xml/pdf)  
**signature_files[]**  Array de arquivos de assinatura 
**detached_files[]**   Array de arquivos assinados - Somente para assinatura detached!  
=====================  ======================================================================

.. note::
  O valor de detached_files[] é respectivamente correspondentes às assinaturas em signature_files[]. Utilize apenas se todas as assinaturas em signature_files[] forem destacadas!

Exemplo de requisição:

.. code-block:: console

		POST 'https://verificador.staging.iti.br/report' \
		--header 'Content-Type: multipart/form-data' \
		--form 'report_type="json"' \
		--form 'signature_files[]=@"/path/to/file/response.p7s"' \
		--form 'detached_files[]=""'


Assinaturas PKCS#7 e PDF
+++++++++++++++++++++++++

Existem duas formas principais de assinar um documento PDF:

* Assinatura *detached*
* Assinatura envelopada

A Assinatura *detached* faz uso de dois arquivos: (1) o arquivo PDF a ser assinado; e (2) um arquivo de assinatura (**.p7s**). Nesta modalidade de assinatura, nenhuma informação referente à assinatura é inclusa no PDF. Toda a informação da assinatura está encapsulada no arquivo (.p7s).
Qualquer alteração no PDF irá invalidar a assinatura contida no arquivo no arquivo (.p7s). Para validar esta modalidade de assinatura, é necessário apresentar para o software de verificação os dois arquivos, PDF e (.p7s).

Para realizar esta modalidade de assinatura pela API de assinatura eletrônica avançada, deve-se calcular o hash sha256 sobre todo o arquivo PDF e enviá-lo através da operação **assinarPKCS7** detalhada no tópico anterior. O arquivo binário retornado como resposta desta operação deve ser salvo com a extensão (.p7s).

A assinatura envelopada, por sua vez, inclui dentro do próprio arquivo PDF o pacote de assinatura PKCS#7. Portanto, não há um arquivo de assinatura separado. Para realizar essa modalidade de assinatura deve-se:

1. Preparar o documento de assinatura
2. Calcular quais os *bytes (bytes-ranges)* do arquivo preparado no passo 1 deverão entrar no computo do hash. Diferentemente da assinatura *detached*, o cálculo do hash para assinatura envelopadas em PDF não é o hash SHA256 do documento original (integral). É uma parte do documento preparado no passo 1.
3. Calcular o hash SHA256 desses *bytes* 
4. Submeter o hash SHA256 à operação **assinarPKCS7** desta API.
5. O resultado da operação **assinarPKCS7** deve ser codificado em hexadecimal e embutido no espaço que foi previamente alocado no documento no passo 1.

O detalhamento de como preparar o documento, calcular os *bytes-ranges* utilizados no computo do hash e como embutir o arquivo PKCS7 no arquivo previamente preparado podem ser encontrados na especificação ISO 32000-1:2008. Existem bibliotecas que automatizam esse procedimento de acordo com o padrão (ex: PDFBox para Java e iText para C#).

Exemplo de aplicação
++++++++++++++++++++

Logo abaixo, encontra-se um pequeno exemplo PHP para prova de conceito.

`Download Exemplo PHP <https://github.com/servicosgovbr/manual-integracao-assinatura-eletronica/raw/main/downloadFiles/exemploApiPhp.zip>`_

Este exemplo é composto por 3 arquivos:

1. index.php -  Formulário para upload de um arquivo
2. upload.php - Script para recepção de arquivo e cálculo de seu hash SHA256. O Resultado do SHA256 é armazenado na sessão do usuário.
3. assinar.php - Implementação do handshake OAuth, assim como a utilização dos dois endpoints acima. Como resultado, uma página conforme a figura abaixo será apresentada, mostrando o certificado emitido para o usuário autenticado e a assinatura.


.. image:: images/image.png


Para executar o exemplo, é possível utilizar Docker com o comando abaixo:

.. code-block:: console
	
		docker-compose up -d

e acessar o endereço http://127.0.0.1:8080

Como criar um par de chaves PGP
+++++++++++++++++++++++++++++++

**GnuPG para Windows** 

Faça o download do aplicativo Gpg4win em: https://gpg4win.org/download.html
O Gpg4win é um pacote de instalação para qualquer versão do Windows, que inclui o software de criptografia GnuPG. Siga abaixo as instruções detalhadas de como gerar um par de chaves PGP:

1. Após o download, execute a instalação e deixe os seguintes componentes marcados conforme imagem abaixo:

.. image:: images/pgp1.png

2. Concluída a instalação, execute o **Kleopatra** para a criação do par de chaves. Kleopatra é uma ferramenta para gerenciamento de certificados X.509, chaves PGP e também para gerenciamento de certificados de servidores. A janela principal deverá se parecer com a seguinte:

.. image:: images/pgp2.png

3. Para criar novo par de chaves (pública e privada), vá até o item do Menu **Arquivo** → **Novo Par de chaves...** selecione **Criar um par de chaves OpenPGP pessoal**. Na tela seguinte informe os detalhes **Nome** e **Email**, marque a opção para proteger a chave com senha e clique em **Configurações avançadas...**

4. Escolha as opções para o tipo do par de chaves e defina uma data de validade. Esta data pode ser alterada depois. Após confirmação da tela abaixo, abrirá uma janela para informar a senha. O ideal é colocar uma senha forte, que deve conter pelo menos 8 caracteres, 1 digito ou caractere especial.

.. image:: images/pgp3.png

5. Após concluído, o sistema permite o envio da chave pública por email clicando em **Enviar chave pública por e-mail...** ou o usuário tem a opção de clicar em **Terminar** e exportar a chave pública para enviá-la por email posteriormente. Para exportar a chave pública e enviá-la anexo ao email, clique com
botão direito na chave criada e depois clique em **Exportar...**

**GnuPG para Linux** 

Praticamente todas as distribuições do Linux trazem o GnuPG instalado e para criar um par de chaves pública e privada em nome do utilizador 'Fulano de Tal', por exemplo, siga os passos abaixo:


1. Abra o terminal e execute o comando abaixo e informe os dados requisitados (Nome e Email). Se não forem especificados os parâmetros adicionais, o tipo da chave será RSA 3072 bits. Será perguntado uma frase para a senha (frase secreta, memorize-a), basta responder de acordo com o que será pedido.

.. code-block:: console

		$ gpg --gen-key
		
		gpg (GnuPG) 2.2.19; Copyright (C) 2019 Free Software Foundation, Inc.
		This is free software: you are free to change and redistribute it.
		There is NO WARRANTY, to the extent permitted by law.
		gpg: directory '/home/user/.gnupg' created
		gpg: keybox '/home/user/.gnupg/pubring.kbx' created
		Note: Use "gpg --full-generate-key" for a full featured key generation dialog.

	    O GnuPG precisa construir uma ID de usuário para identificar sua chave.

		Nome completo: **Fulano de Tal**
		Endereço de correio eletrônico: **fulanodetal@email.com**
		Você selecionou este identificador de usuário: "Fulano de Tal <fulanodetal@email.com>"
		Change (N)ame, (E)mail, or (O)kay/(Q)uit? O

		gpg: /home/user/.gnupg/trustdb.gpg: banco de dados de confiabilidade criado
        gpg: chave D5882F501CC722AA marcada como plenamente confiável
        gpg: directory '/home/user/.gnupg/openpgp-revocs.d' created
        gpg: revocation certificate stored as '/home/user/.gnupg/openpgprevocs.d/269C3D6B65B150A9B349170D5882F501CC722AA.rev'

		Chaves pública e privada criadas e assinadas.

		pub rsa3072 2021-04-30 [SC] [expira: 2023-04-30] 269C3D6B65B150A9B349170D5882F501CC722AA uid Fulano de Tal <fulanodetal@email.com>
        sub rsa3072 2021-04-30 [E] [expira: 2023-04-30]
		
2. Para enviar um documento ou um e-mail cifrado com sua chave, é necessário que a pessoa tenha a sua chave pública. Partindo do ponto que a pessoa fez um pedido da sua chave pública, então é necessário criar um arquivo
com a chave e passar o arquivo para o solicitante (por exemplo, podemos passar pelo e-mail). Execute o comando abaixo no terminal do Linux para exportar a sua chave para o arquivo **MinhaChave.asc**

.. code-block:: console
	
		$ gpg --export 269C3D6B65B150A9B449170D5882F501CC722AA> MinhaChave.asc

A sequência de números e letras "269C3D6B65B150A9B349170D5882F501CC722AA" é o ID da chave (da chave que criamos aqui no exemplo, substitua pelo seu ID) e **MinhaChave.asc** é o nome do arquivo onde será gravada a chave (pode ser outro nome).
O próximo passo é o envio do arquivo com a chave pública para a pessoa e então ela poderá criptografar um e-mail ou um documento com a sua chave pública. Se foi criptografado com a sua chave pública, somente a sua chave privada será capaz de decodificar o documento e a frase secreta de sua chave será requisitada.

3. Para **decifrar** um documento que foi criptografado com a sua chave pública basta seguir os passos abaixo, substituindo **NomeArquivo.gpg** pelo nome do arquivo cifrado. Será solicitada a frase secreta de sua chave privada. Um arquivo com nome **ArquivoTextoClaro** será criado na mesma pasta. Este arquivo contêm as informações decifradas.		

.. code-block:: console
	
		$ gpg -d NomeArquivo.gpg > ArquivoTextoClaro

		gpg: criptografado com 3072-bit RSA chave, ID 4628820328759F85, criado 2021-04-24 "Fulano de Tal <fulanodetal@email.com>"






.. |site externo| image:: images/site-ext.gif
.. _`codificador para Base64`: https://www.base64decode.org/
.. _`Plano de Integração`: arquivos/Modelo_PlanodeIntegracao_LOGINUNICO_final.doc
.. _`OpenID Connect`: https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse
.. _`auth 2.0 Redirection Endpoint`: https://tools.ietf.org/html/rfc6749#section-3.1.2
.. _`Exemplos de Integração`: exemplointegracao.html
.. _`Design System do Governo Federal`: http://dsgov.estaleiro.serpro.gov.br/ds/componentes/button
.. _`Resultado Esperado do Acesso ao Serviço de Confiabilidade Cadastral (Selos)`: iniciarintegracao.html#resultado-esperado-do-acesso-ao-servico-de-confiabilidade-cadastral-selos
.. _`Resultado Esperado do Acesso ao Serviço de Confiabilidade Cadastral (Categorias)` : iniciarintegracao.html#resultado-esperado-do-acesso-ao-servico-de-confiabilidade-cadastral-categorias
.. _`Documento verificar Código de Compensação dos Bancos` : arquivos/TabelaBacen.pdf
.. _`Login Único`: https://manual-roteiro-integracao-login-unico.servicos.gov.br/pt/stable/index.html
.. _`Lei n° 14.063`: http://www.planalto.gov.br/ccivil_03/_ato2019-2022/2020/lei/L14063.htm
.. _`SEDGGME Nº 2.154/2021`: https://www.in.gov.br/web/dou/-/portaria-sedggme-n-2.154-de-23-de-fevereiro-de-2021-304916270
