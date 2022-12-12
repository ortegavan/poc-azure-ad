![](https://github.com/ortegavan/poc-azure-ad/blob/958c31908171220166a69d79819276b71d4a3d21/README.jpg)

<h2 align="center">
    Autenticando usuários no Azure AD via aplicação Angular
</h2>
<p align="center">
    <a href="https://github.com/ortegavan/poc-azure-ad/commits/">
        <img alt="GitHub last commit" src="https://img.shields.io/github/last-commit/ortegavan/poc-azure-ad?style=flat-square">
    </a>
    <a href="https://github.com/prettier">
        <img alt="code style: prettier" src="https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square">
    </a>   
    <a href="https://github.com/ortegavan">
        <img alt="Made with love" src="https://img.shields.io/badge/made%20with%20%E2%99%A5%20by-ortegavan-ff69b4.svg?style=flat-square">
    </a>
</p>

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
    "url": "http://localhost:4200/profile",
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
				redirectUri: 'http://localhost:4200/profile',
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
import { Component, OnInit, OnDestroy } from '@angular/core';
import { MsalService, MsalBroadcastService } from '@azure/msal-angular';
import { InteractionStatus } from '@azure/msal-browser';
import { Subject } from 'rxjs';
import { filter, takeUntil } from 'rxjs/operators';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent implements OnInit, OnDestroy {
  title = 'msal-angular-tutorial';
  isIframe = false;
  loginDisplay = false;
  private readonly _destroying$ = new Subject<void>();

  constructor(private broadcastService: MsalBroadcastService, private authService: MsalService) { }

  ngOnInit() {
    this.isIframe = window !== window.parent && !window.opener;

    this.broadcastService.inProgress$
    .pipe(
      filter((status: InteractionStatus) => status === InteractionStatus.None),
      takeUntil(this._destroying$)
    )
    .subscribe(() => {
      this.setLoginDisplay();
    })
  }

  login() {
    this.authService.loginRedirect();
  }

  setLoginDisplay() {
    this.loginDisplay = this.authService.instance.getAllAccounts().length > 0;
  }

  ngOnDestroy(): void {
    this._destroying$.next(undefined);
    this._destroying$.complete();
  }
}
```

Nosso home.component.ts do exemplo verifica se o usuário está autenticado, e o código ficou assim:

```javascript
import { Component, OnInit } from '@angular/core';
import { MsalBroadcastService, MsalService } from '@azure/msal-angular';
import { EventMessage, EventType, InteractionStatus } from '@azure/msal-browser';
import { filter } from 'rxjs/operators';

@Component({
  selector: 'app-home',
  templateUrl: './home.component.html',
  styleUrls: ['./home.component.css']
})
export class HomeComponent implements OnInit {
  loginDisplay = false;

  constructor(private authService: MsalService, private msalBroadcastService: MsalBroadcastService) { }

  ngOnInit(): void {
    this.msalBroadcastService.msalSubject$
      .pipe(
        filter((msg: EventMessage) => msg.eventType === EventType.LOGIN_SUCCESS),
      )
      .subscribe((result: EventMessage) => {
        console.log(result);
      });

    this.msalBroadcastService.inProgress$
      .pipe(
        filter((status: InteractionStatus) => status === InteractionStatus.None)
      )
      .subscribe(() => {
        this.setLoginDisplay();
      })
  }

  setLoginDisplay() {
    this.loginDisplay = this.authService.instance.getAllAccounts().length > 0;
  }
}
```

E o home.component.html agora tem um código condicional e ficou assim: 

```javascript
<div *ngIf="!loginDisplay">
    <p>Please sign-in to see your profile information.</p>
</div>

<div *ngIf="loginDisplay">
    <p>Login successful!</p>
    <p>Request your profile information by clicking Profile above.</p>
</div>
```

### Usando o token

O MSAL fornece um interceptor que automaticamente recupera o token para ser utilzado em outras chamadas. O app.module.ts precisa estar configurado. Comece importando os módulos para interceptação:

```javascript
import { HTTP_INTERCEPTORS, HttpClientModule } from "@angular/common/http";
```

Complete os imports do MSAL:

```javascript
import { MsalModule, MsalRedirectComponent, MsalGuard, MsalInterceptor } from '@azure/msal-angular';
import { InteractionType, PublicClientApplication } from '@azure/msal-browser';
```

Complete o arquivo: 

```javascript
@NgModule({
  declarations: [AppComponent, HomeComponent, ProfileComponent],
  imports: [
    BrowserModule,
    AppRoutingModule,
    NoopAnimationsModule,
    MatButtonModule,
    MatToolbarModule,
    MatListModule,
    HttpClientModule,
    MsalModule.forRoot(
      new PublicClientApplication({
        auth: {
          clientId: '1bde1000-2c3a-4bc6-8d8c-93f4ce8205c8',
          authority: 'https://login.microsoftonline.com/dc1e6c52-6944-4171-a933-38a16b9dc72b',
          redirectUri: 'http://localhost:4200/home',
        },
        cache: {
          cacheLocation: 'localStorage',
          storeAuthStateInCookie: false,
        },
      }),
      {
        interactionType: InteractionType.Redirect,
        authRequest: {
          scopes: ['user.read'],
        },
      },
      {
        interactionType: InteractionType.Redirect,
        protectedResourceMap: new Map([
          ['https://graph.microsoft.com', ['user.read']],
        ]),
      }
    ),
  ],
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: MsalInterceptor,
      multi: true,
    },
    MsalGuard,
  ],
  bootstrap: [AppComponent, MsalRedirectComponent],
})
export class AppModule {}
```

Como exemplo, aqui criamos o componente Profile que, agora, tem seu HTML assim:

```javascript
<div>
    <p><strong>First Name: </strong> {{profile?.givenName}}</p>
    <p><strong>Last Name: </strong> {{profile?.surname}}</p>
    <p><strong>Email: </strong> {{profile?.userPrincipalName}}</p>
    <p><strong>Id: </strong> {{profile?.id}}</p>
</div>
```

E o profile.component.ts ficou assim:

```javascript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

const GRAPH_ENDPOINT = 'https://graph.microsoft.com/v1.0/me';

type ProfileType = {
  givenName?: string;
  surname?: string;
  userPrincipalName?: string;
  id?: string;
};

@Component({
  selector: 'app-profile',
  templateUrl: './profile.component.html',
  styleUrls: ['./profile.component.css'],
})
export class ProfileComponent implements OnInit {
  profile!: ProfileType;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.getProfile();
  }

  getProfile() {
    this.http.get(GRAPH_ENDPOINT).subscribe((profile) => {
      this.profile = profile;
    });
  }
}
```

Na index.html, abaixo do app-root, adiciona:

```javascript
<app-redirect></app-redirect>
```

### Sign out

Para se deslogar na aplicação Angular, simplesmente adicione o método abaixo no botão de sair:

```javascript
logout() {
    this.authService.logoutPopup({
      mainWindowRedirectUri: '/',
    });
  }
```

### Referência 

[https://learn.microsoft.com/en-us/azure/active-directory/develop/tutorial-v2-angular-auth-code](https://learn.microsoft.com/en-us/azure/active-directory/develop/tutorial-v2-angular-auth-code)