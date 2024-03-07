- [[#1. Código Node.js]]
- [[#2. Configurando N8N]]
- [[#2.1. Rodar o código]]
- [[#3. Teste]]
# 1. Código Node.js
- Crie uma pasta para o código que será utilizado. 
- Instale as dependências necessárias: 
```bash
npm install @wppconnect-team/wppconnect
npm install express
npm install http
npm init
```

- Crie o arquivo `app-node.js`, que irá conter o código do WPPConnect, que é uma implementação robusta de um bot de WhatsApp, destacando-se por sua capacidade de escutar, encaminhar e enviar mensagens programáticas. 
```javascript
const wppconnect = require('@wppconnect-team/wppconnect')
const express = require('express');
const app = express()
const http = require('http');
const server = http.createServer(app);
const port = 8051;
const request = require('request');

app.use(express.json())
app.use(express.urlencoded({extended: true}));

wppconnect.create({
	session: 'sessionName',
	statusFind: (statusSession, session) => {
		console.log('Status Session: ', statusSession);
		console.log('Session name: ', session);
	},
	headless: true,
	puppeteerOptions: {
		userDataDir: 'tokens/sessionName',
		headless: "new"
	},
})
.then((client) => start(client))
.catch((error) => {
		console.error('Erro ao criar a sessão WPPConnect:', error);
		process.exit(1);
});

async function start(client) {
	console.log('BOT-ZDG - Iniciando o BOT...');
	
	//escuta mensagem
	client.onMessage(async (msg) => {
		try{
			const options = {
				'method': 'POST',
				'url': 'http://localhost:5678/webhook-test/304535cc-f03c-490e-8436-776493330b6d',
				'headers': {
					'Content-Type': 'application/json',
					'teste': '12345'
				},
				json: msg
			};
			request(options, function (error, response) {
				if (error) {
					throw new Error(error);
				}
				else {
					console.log(response.body);
				}
			});
		} catch(e) {
			console.log(e);
		}
	});
	
	//envia mensagem
	app.post('/send-message', async(req, res) => {
		const number = req.body.number;
		const message = req.body.message;
		client.sendText(number, message).then(response => {
			res.status(200).json({
				status: true,
				message: 'BOT-ZDG Mensagem enviada',
				response: response
			});
		}).catch(err => {
			res.status(500).json({
				status: false,
				message: 'BOT-ZDG Mensagem não enviada',
				response: err.text
			});
		});
	});
}

server.listen(port, function() {
	console.log('BOT-ZDG rodando na porta*: ' + port);
});


```

O código acima:
```txt
- Inicializa um servidor express com o nome de sessão "sessionName" e define a porta 8051 para escutar as requisições.
- Utiliza a biblioteca `wppconnect` para criar um cliente WhatsApp, configurando opções como o nome da sessão, callback para status da sessão e opções do Puppeteer para um navegador headless.

- Implementa um callback para lidar com eventos de alteração de status da sessão, exibindo informações relevantes no console durante a execução do bot.

- Configura um listener para mensagens recebidas (`client.onMessage`), que encaminha a mensagem para uma URL externa por meio de uma requisição POST utilizando a biblioteca `request`.
- Demonstra a capacidade do bot de integrar-se com outros sistemas ou serviços, proporcionando flexibilidade na manipulação de mensagens recebidas.

- Estabelece uma rota HTTP (`/send-message`) para receber solicitações POST, permitindo o envio programático de mensagens ao WhatsApp.
- Essa funcionalidade oferece uma interface para a automação de interações no WhatsApp, possibilitando a integração com outros aplicativos e sistemas.

- Inicia o servidor express na porta 8051, indicando no console que o bot está em execução e pronto para interações automatizadas no WhatsApp.
```


# 2. Configurando N8N

### Node Webhook
- Crie um node webhook. 
	- O `HTTP Method` deve ser `POST`.
- Copie o link do webhook e adicione na URL do código acima.

- Passa prosseguir com o desenvolvimento do workflow, rode o código acima ([[#1. Código Node.js]]), conforme demonstrado em [[#3. Rodar o código]]. Esse passo é necessário para que os parâmetros que serão utilizados possam ser reconhecidos pelo N8N. 

### Node If
- Crie um node if
	- Adicione uma condição. Selecione uma condição do tipo `string`, e `is equal to` 
	- Primeiro defina a variável que vai ser comparada: 
		- No item `Conditions`, defina para `expression`, e clique na setinha no canto inferior direito para `add expression`.
		- Em `Variable Selector`, selecione `Nodes` -> `Webhook` -> `Output Data` -> `JSON` -> `body` -> `body`. A expressão final ficará assim: `{{$node["Webhook"].json["body"]["body"]}}`
	- Após isso, defina a string que será utilizada na comparação. No nosso caso, é `Oi`

### If False -> Node httpRequest
- Crie um node `httpRequest` como resultado para `true` do `If`
	- Defina o `method` para `POST`
	- Defina a `URL`. No nosso código, foi definida a porta 8051`localhost`, e na foi definido um método para envio de mensagens do WppConnect no endpoint `/send-message`. Assim, o nosso `URL` é: `http://localhost:8051/send-message`.
	- Selecione o parâmetro `Send Body`
		- Em `Body Content Type`, selecione `JSON`
		- Em `Specify Body`, selecione `Using Fields Below`
			- No primeiro parâmetro:
				- Defina `Name` para "number"
				- Em `Value`, selecione `Expression`, e clique na setinha no canto inferior direito para `add expression`.
				- Em `Variable Selector`, selecione `Nodes` -> `Webhook` -> `Output Data` -> `JSON` -> `body` -> `sender` -> `from`. A expressão final ficará assim: `{{$["Webhook"].item.json["body"]["from"]}}`
			- No segundo parâmetro:
				- Defina `Name` para "message"
				- Defina `Value` para "Olá true"

> No node `httpRequest`, definimos o envio de dois parâmetros (number, message) para um endpoint especificado, no caso de a condição anterior ser verdadeira. 
> Observe que esses dois parâmetros foram definidos no código [[#1. Código Node.js]], e que são os parâmetros para a chamada de uma função, ao chamar uma requisição `POST` no endpoint `/send-message`

### If False -> Node httpRequest

Esse node é quase igual ao anterior.

- Crie um node `httpRequest` como resultado para `true` do `If`
	- Defina o `method` para `POST`
	- Defina a `URL`. No nosso código, foi definida a porta 8051`localhost`, e na foi definido um método para envio de mensagens do WppConnect no endpoint `/send-message`. Assim, o nosso `URL` é: `http://localhost:8051/send-message`.
	- Selecione o parâmetro `Send Body`
		- Em `Body Content Type`, selecione `JSON`
		- Em `Specify Body`, selecione `Using Fields Below`
			- No primeiro parâmetro:
				- Defina `Name` para "number"
				- Em `Value`, selecione `Expression`, e clique na setinha no canto inferior direito para `add expression`.
				- Em `Variable Selector`, selecione `Nodes` -> `Webhook` -> `Output Data` -> `JSON` -> `body` -> `sender` -> `from`. A expressão final ficará assim: `{{$["Webhook"].item.json["body"]["from"]}}`
			- No segundo parâmetro:
				- Defina `Name` para "message"
				- Defina `Value` para "A gente diz oi quando chega"

> No node `httpRequest`, definimos o envio de dois parâmetros (number, message) para um endpoint especificado, no caso de a condição anterior ser verdadeira. 
> Observe que esses dois parâmetros foram definidos no código [[#1. Código Node.js]], e que são os parâmetros para a chamada de uma função, ao chamar uma requisição `POST` no endpoint `/send-message`

# 2.1. Rodar o código
- Rode o código. Use `node app-n8n.js`
- Abra o Whatsapp, vá em `Dispositivos Conectados`, e clique em `Adicionar Dispositivo`.
- Leia o QR Code que foi gerado no `cmd` quando o código terminou de rodar 

# 3. Teste
- Após terminar a implementação do código e do workflow, faça um teste. 
- Envie a mensagem "Oi" de um outro número para o celular que você usou. O resultado esperado é que o bot responsa "Oi true".
- Envie qualquer outra mensagem de um outro número para o celular que você usou. O resultado esperado é que o bot responsa "Oi false"