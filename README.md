![](https://github.com/ortegavan/poc-azure-ad/blob/958c31908171220166a69d79819276b71d4a3d21/README.jpg)

## Autenticando usuários no Azure AD via aplicação Angular

### Instale a Microsoft Authentication Library (MSAL) na aplicação Angular. É necessário instalar a biblioteca e o módulo de autenticação via browser:

```
npm install @azure/msal-browser
npm install @azure/msal-angular
```

### Registre a aplicação no Azure Active Directory:

- Acesse o portal do Azure e localize o Azure Active Diretory;
- Clique em Adicionar, depois em Registro do aplicativo;
- Informe um nome para a aplicação, você pode alterá-lo depois, se precisar;
- Em Tipos de conta, selecione a primeira opção: Único locatário;
- Não preencha nada na URI de redirecionamento e clique em Registrar.

Salve o ID do aplicativo cliente para usar depois.

### Habilite e configure a URI de redirecionamento:

**Para ambiente de dev:**

- Na página de Visão geral do aplicativo registrado, clique em Manifesto, no menu à esquerda;
- Localize o atributo replyUrlsWithType e cadastre a URI de dev, no nosso exemplo é:

```javascript
{
    "url": "http://localhost:4200/home",
    "type": "Spa"
}
```

**Para ambiente de produção:**

- Na página de Visão geral do aplicativo registrado, clique em Adicionar URIs de redirecionamento;
- Selecione SPA;
- Informe a URI, não marque nenhum dos checkboxes seguintes e clique em Configure.

### De volta à aplicação Angular, importe e registre os módulos do MSAL no app.module.ts:

```javascript
import { MsalModule } from '@azure/msal-angular';
import { PublicClientApplication } from '@azure/msal-browser';
```

Em clientId informe o ID do aplicativo cliente que aparece na página de Visão geral do aplicativo registrado no Azure; em authority informe a URL de autenticação da Microsoft e o ID do locatário do Azure, no nosso exemplo é:

```javascript
imports: [
	BrowserModule,
	AppRoutingModule,
	MsalModule.forRoot( new PublicClientApplication({
		auth: {
				clientId: '1bde1000-2c3a-4bc6-8d8c-93f4ce8205c8',
				authority: 'https://login.microsoftonline.com/dc1e6c52-6944-4171-a933-38a16b9dc72b',
				redirectUri: 'http://localhost:4200/home',
			},
			cache: {
				cacheLocation: 'localStorage',
				storeAuthStateInCookie: true,
			}
		}
	), null, null)
],
```

### Faça as alterações a seguir no arquivo de routing:

No app-routing.module.ts, adicione a constante isIframe que será usada posteriormente:

```javascript
const isIframe = window !== window.parent && !window.opener;
```

No RouterModule.forRoot, inclua a propriedade initialNavigation:

```javascript
RouterModule.forRoot(routes, {
	initialNavigation:
		!BrowserUtils.isInIframe() && !BrowserUtils.isInPopup()
		? 'enabledNonBlocking'
		: 'disabled',
}),
```

### Proceda com as alterações no componente de login existente onde está o botão para se autenticar, no nosso exemplo é o app.component:

Em app.component.html, altere para que o router-outlet fique assim:

```javascript
<router-outlet *ngIf="!isIframe"></router-outlet>
```

Em app.component.ts, importe o MsalService:

```javascript
import { MsalService } from '@azure/msal-angular';
```

Declare as variáveis:

```javascript
isIframe = false;
loginDisplay = false;
```

No construtor, injete o MsalService:

```javascript
constructor(private authService: MsalService) {}
```

No ngOnInit, faça as alterações:

```javascript
ngOnInit() {
	this.isIframe = window !== window.parent && !window.opener;
}
```

No método de login a ser chamado pelo botão de autenticação, faça as alterações:

```javascript
login() {
	this.authService.loginPopup().subscribe({
		next: (result) => {
			console.log(result);
			this.setLoginDisplay();
		},
		error: (error) => console.log(error),
	});
}
```

Por fim, crie o método setLoginDisplay:

```javascript
setLoginDisplay() {
	this.loginDisplay = this.authService.instance.getAllAccounts().length > 0;
}
```

### Exibindo um conteúdo após autorização

Adicione o MsalBroadcastService no app.component.ts e faça um subscribe ao inProgress$, seu código deve ficar assim:

```javascript

```