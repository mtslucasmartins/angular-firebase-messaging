# Adicionar Notificações com FCM
---
## Bibliotecas Usadas

### Neste pojeto vamos usar as bibliotecas *firebase* and *@angular/fire*. Vamos instalá-las. 
```shell
  $ npm install firebase @angular/fire --save
````
---
### No arquivo 'src/manifest.json', vamos adicionar a linha:
```json
{
  "gcm_sender_id": "<SenderID>", 
   ...
}
```
>  `<SenderID>` será o **Código do remetente** (Sender ID) da  nossa aplicação no Firebase.
---
## Criando a Service Worker do Firebase FCM.
### Vamos criar o arquivo 'src/firebase-messaging-sw.js' com o seguinte conteúdo
```javascript
	// Give the service worker access to Firebase Messaging.
	// Note that you can only use Firebase Messaging here, other Firebase libraries
	// are not available in the service worker.
	importScripts('https://www.gstatic.com/firebasejs/5.5.0/firebase-app.js');
	importScripts('https://www.gstatic.com/firebasejs/5.5.0/firebase-messaging.js');

	// Initialize the Firebase app in the service worker by passing in the
	// messagingSenderId.
	firebase.initializeApp({
	  'messagingSenderId': '<SenderID>'
	});

	// Retrieve an instance of Firebase Messaging so that it can handle background
	// messages.
	const messaging = firebase.messaging();
```


---


## Criando um Provider para a aplicação.

### Vamos criar um arquivo 'src/app/shared/messaging.service.ts'.

```typescript
import { Injectable } from '@angular/core';
import { mergeMapTo } from 'rxjs/operators';
import { take } from 'rxjs/operators';
import { AngularFireMessaging } from '@angular/fire/messaging';
import { BehaviorSubject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class MessagingService {

  currentMessage = new BehaviorSubject(null);

  constructor(private fcm: AngularFireMessaging) {
    this.fcm.messaging.subscribe((messaging: any) => {
      messaging.onMessage = messaging.onMessage.bind(messaging);
      messaging.onTokenRefresh = messaging.onTokenRefresh.bind(messaging);
    });
  }

  /**
   * request permission for notification from firebase cloud messaging
   * 
   * @param userId userId
   */
  public requestPermission(userId) {
    this.fcm.requestToken.subscribe((token: any | string) => {
      console.log(token);
			// lógica para salvar o token e o id do usuário para envio de notificações
    }, (err: any) => {
      console.error('Unable to get permission to notify.', err);
    });
  }

  /**
   * hook method when new notification received in foreground
   */
  receiveMessage() {
    this.fcm.messages.subscribe((payload) => {
      console.log('new message received. ', payload);
      this.currentMessage.next(payload);
    });
  }
}
```

> **requestPermission()** : Navegador / Dispositivo pergunta ao usuário por permissão para receber notificações. Após permissão concedida pelo usuário, firebase vai retornar um **token**, que usamos como referência a esse usuário para enviar notificações.

> **receiveMessage()** : Função executada quando uma notificação é recebida.


---


## Configurando nossa aplicação.

### Antes de conseguirmos iniciar nossa aplcação precisamos configurar algumas variaveis de anmbiente e adicionar alguns assets.

```json
/**
 * environment.ts
 */
export const environment = {
  production: false,
  firebase: {
    apiKey: "your apikey",
    authDomain: "your authDomain",
    databaseURL: "your databaseUrl",
    projectId: "your projectId",
    storageBucket: "your storageBucket",
    messagingSenderId: "your messagingSenderId"
  }
};
```

### Agora precisamos adicionar o nosso arquivo 'firebase-messaging-sw.js' no angular.json, para ter certeza de que esse arquivo será incluído quando executarmos ou fizermos da nossa aplicação.

```json
"build":{
   "builder":"@angular-devkit/build-angular:browser",
   "options":{
      ...
      "assets":[
         "src/favicon.ico",
         "src/assets",
         "src/firebase-messaging-sw.js", // <-- adicione 
         "src/manifest.json" 
      ],
      "styles":[
         "src/styles.css"
      ],
      "scripts":[

      ]
   }
...
}
```

---
## Implementação

### Finanlmente, nosso código está pronto para ser usado. 
```typescript
import { Component } from '@angular/core';
import { MessagingService } from "./shared/messaging.service";
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  
  message;

  constructor(private messagingService: MessagingService) { }

  ngOnInit() {
    const userId = 'user001';
    this.messagingService.requestPermission(userId)
    this.messagingService.receiveMessage()
    this.message = this.messagingService.currentMessage
  }
}
```

### No nosso html podemos usar o pipe async e json.
```html
{{ message | async | json }}
```

### app.module.ts
```typescript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { ServiceWorkerModule } from '@angular/service-worker';
import { environment } from '../environments/environment';
import { AsyncPipe } from '@angular/common';
import { AngularFireMessagingModule } from '@angular/fire/messaging';
import { AngularFireModule } from '@angular/fire';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    AppRoutingModule,
    ServiceWorkerModule.register('ngsw-worker.js', { enabled: environment.production }),
    AngularFireMessagingModule,
    AngularFireModule.initializeApp(environment.firebase),
  ],
  providers: [AsyncPipe],
  bootstrap: [AppComponent]
})
export class AppModule { }

```