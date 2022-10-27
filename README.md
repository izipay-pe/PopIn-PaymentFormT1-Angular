# PopIn-PaymentFormT1-Angular

Esta página explica cómo crear un formulario de pago dinámico desde cero utilizando Angular y angular-cli y la biblioteca de embedded-form-glue.

<p align="center">
  <img src="/image/imagenes-readme/imagen-popin.png?raw=true" alt="Formulario"/>
</p> 

<a name="Requisitos_Previos"></a>

## Requisitos Previos

* Extraer credenciales del Back Office Vendedor. [Guía Aquí](https://github.com/izipay-pe/obtener-credenciales-de-conexion)

* Debe instalar la [versión de LTS node.js](https://nodejs.org/es/).


## 1.- Crear el proyecto

* Descargar el proyecto .zip haciendo click [Aquí](https://github.com/izipay-pe/PopIn-PaymentFormT1-Angular/archive/refs/heads/main.zip) o clonarlo desde Git.  
```sh
git clone https://github.com/izipay-pe/PopIn-PaymentFormT1-Angular.git
``` 

* Ingrese a la carpeta raiz del proyecto.


* A continuación, instale el cliente angular-cli:

```bash
npm install -g @angular/cli
```

Más detalles en la página web de [angular-cli](https://angular.io/guide/quickstart).

* Agregue la dependencia con:

```bash
npm install --save @lyracom/embedded-form-glue
```

Ejecútelo y pruébelo en minimal-example:

```sh
npm run start
```

ver el resultado en http://localhost:4200/

## 2.- Agregar el formulario de pago

**Nota**: Reemplace **[CHANGE_ME]** con sus credenciales de `API REST` extraídas desde el Back Office Vendedor, ver [Requisitos Previos](#Requisitos_Previos).

* Editar en src/index.html en la sección HEAD.

```javascript
<!-- tema y plugins. debe cargarse en la sección HEAD -->
<link rel="stylesheet"
href="~~CHANGE_ME_ENDPOINT~~/static/js/krypton-client/V4.0/ext/classic-reset.css">
<script
    src="~~CHANGE_ME_ENDPOINT~~/static/js/krypton-client/V4.0/ext/classic.js">
</script>
```

* Edita src/app/app.component.html template, se agregará al elemento #myPaymentForm para ver formulario de pago.

```html
<div class="form">
  <h1>{{ title }}</h1>
  <div class="container">
    <div id="myPaymentForm"></div>
  </div>
</div>
```

* Actualice los estilos en src/app/app.component.css.

```css
h1 {
  margin: 40px 0 20px 0;
  width: 100%;
  text-align: center;
}
.container {
  display: flex;
  justify-content: center;
}
```

* Edite el componente predeterminado src/app/app.component.ts, con el siguiente codigo si quiere interactuar con el formulario de pago, con un endpoint propio.

```js
import { HttpClient,HttpHeaderResponse,HttpHeaders } from "@angular/common/http";
import { Component, AfterViewInit, ChangeDetectorRef} from "@angular/core";
import KRGlue from "@lyracom/embedded-form-glue";
import { firstValueFrom } from "rxjs";

@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.css"]
})
export class AppComponent implements AfterViewInit{
  title: string = "Ejemplo de un formulario popin en ANGULAR";
  message: string = "";

  constructor(
    private http: HttpClient,
    private chRef: ChangeDetectorRef
  ){}

  ngAfterViewInit(){

    /* llenar con las credenciales extraídas del Back Office Vendedor para mas detalle regresar a:  Requisitos Previos. */

    const endpoint = "~~CHANGE_ME_ENDPOINT~~";
    const publicKey = "~~CHANGE_ME_PUBLIC_KEY~~";
    const formToken = ""; /* solo está declarando la variable dejar vacío */

    /* variable que recibe el monto a pagar */
    let monto = 80;

    /* arreglo que enviara la data al endpoint */
    const data = {
                  amount: monto*100,
                  currency: 'PEN'
    };

    /* abre una nueva conexión, utilizando la solicitud POST en el URL de su endpoint */
    const observable = this.http.post("YOUR_SERVER/payment/init", data,{responseType: 'text'});

    firstValueFrom(observable).then((resp: any) => {
      formToken = resp
      return KRGlue.loadLibrary(endpoint, publicKey) /* cargar la libreria KRGlue */
    })
    .then(({ KR }) =>
      KR.setFormConfig({
        /* establecer la configuración mínima */
        formToken: formToken,
        "kr-language": "es-ES" /* cambia el idioma del formulario */
      })
    )
    .then(({ KR }) => KR.onSubmit(this.onSubmit))
    .then(({ KR }) => KR.addForm("#myPaymentForm")) /* agregar un formulario de pago a myPaymentForm div */
    .then(({ KR, result }) => KR.showForm(result.formId)) /* muestra el formulario de pago */
    .catch(
      error => {
        this.message = error.message + " (see console for more details)";
      }
    );
  }
}
```

    2.1.- El hash de pago debe validarse en el lado del servidor para evitar la exposición de su clave hash personal.

    En el lado del servidor:

    ```js
    const express = require('express')
    const hmacSHA256 = require('crypto-js/hmac-sha256')
    const Hex = require('crypto-js/enc-hex')
    const app = express()
    (...)
    // válida los datos de pago dados (hash)
    app.post('/validatePayment', (req, res) => {
      const answer = req.body.clientAnswer
      const hash = req.body.hash
      const answerHash = Hex.stringify(
        hmacSHA256(JSON.stringify(answer), 'CHANGE_ME: HMAC SHA256 KEY')
      )
      if (hash === answerHash) res.status(200).send('Valid payment')
      else res.status(500).send('Payment hash mismatch')
    })
    (...)
    ```

    Del lado del cliente:

    ```js
    import { Component, OnInit } from "@angular/core";
    import KRGlue from "@lyracom/embedded-form-glue";
    import axios from 'axios'
    @Component({
      selector: "app-root",
      templateUrl: "./app.component.html",
      styleUrls: ["./app.component.css"]
    })
    export class AppComponent implements OnInit {
      title: string = "Ejemplo de un formulario popin en ANGULAR";
      (...),
        ngOnInit() {
          /* use su endpoint y la clave public_key */
          const endpoint = '~~CHANGE_ME_ENDPOINT~~'
          const publicKey = '~~CHANGE_ME_PUBLIC_KEY~~'
          const formToken = 'DEMO-TOKEN-TO-BE-REPLACED'
          KRGlue.loadLibrary(endpoint, publicKey) /* cargar la libreria KRGlue */
            .then(({KR}) => KR.setFormConfig({  /* establecer la configuración mínima */
              formToken: formToken,
              'kr-language': 'en-US',
            })) /* para actualizar el parámetro de inicialización */
            .then(({KR}) => KR.onSubmit(resp => {
              axios
                .post('http://localhost:3000/validatePayment', paymentData)
                .then(response => {
                  if (response.status === 200) this.message = 'Payment successful!'
                })
              return false
            }))
            .then(({KR}) => KR.addForm('#myPaymentForm')) /* crear un formulario de pago */
            .then(({KR, result}) => KR.showForm(result.formId));  /* muestra el formulario de pago */
        }
        (...)
    }
    ```


## 3.- Transacción de prueba

El formulario de pago está listo, puede intentar realizar una transacción utilizando una tarjeta de prueba con la barra de herramientas de depuración (en la parte inferior de la página).

Si intenta pagar, tendrá el siguiente error: **CLIENT_998: Demo form, see the documentation**.
Es porque el **formToken** que ha definido usando **KR.setFormConfig** está configurado en **DEMO-TOKEN-TO-BE-REPLACED**.

you have to create a **formToken** before displaying the payment form using Charge/CreatePayment web-service.
For more information, please take a look to:

- [Formulario incrustado: prueba rápida](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/javascript/quick_start_js.html)
- [Primeros pasos: pago simple](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/javascript/guide/start.html)
- [Servicios web - referencia de la API REST](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/api/reference.html)

## 4.- Implementar IPN

* Ver manual de implementacion de la IPN [Aquí](https://secure.micuentaweb.pe/doc/es-PE/rest/V4.0/kb/payment_done.html)

* Ver el ejemplo de la respuesta IPN [Aquí](https://github.com/izipay-pe/Redirect-PaymentForm-IpnT1-PHP)
