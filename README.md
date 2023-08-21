# DTE Bearer Token (A tener en cuenta)

Cada vez que intente enviar una solicitud de aprobación a la api del DTE (Test o Producción), usted necesitará tener en su poder un token de autorización [Bearer Token](https://programmerclick.com/article/87581985934/#google_vignette).

El token proporcionado por DTE tiene una vigencia de 24 horas desde la última vez que se creo, por lo que es necesario renovarlo cuando la petición no es aceptada.

```json
{
 "authorities": [
    "USER",
    "USER_API",
    "Usuario"
  ],
  "iat": 1692625875,
  "exp": 1692712275
}
```

La llave `"ext"` tiene un formato unix timestamp ([puede convertirlo desde aquí](https://www.epochconverter.com/)). El valor `1692712275` es igual a (expira): `martes, 22 de agosto de 2023 7:51:15`

A continuación, voy a proponer una solución simple pero efectiva en lenguaje python para tener este tema cubierto.

**Importante:**
`Esta solución no toma en cuenta el error en la conexión desde cualquiera de las partes. Usaré las url del ambiente de pruebas`

# Librerías aplicadas

| Librería | README | Objetivo |
| ------ | ------ | ------ |
| python-dotenv | [https://saurabh-kumar.com/python-dotenv/](https://saurabh-kumar.com/python-dotenv/) | Para leer valores desde un archivo `.env` |
| requests | [https://requests.readthedocs.io/en/latest/](https://requests.readthedocs.io/en/latest/) | Para hacer peticiones `HTTP` |
| json | [https://docs.python.org/3/library/json.html](https://docs.python.org/3/library/json.html) | Para decodificar la respuesta en formato `json` |
| pathlib | [https://docs.python.org/3/library/pathlib.html](https://docs.python.org/3/library/pathlib.html) | Para acceder al sistema de archivos  `windows/linux/mac`|

**Para este ejemplo es necesario contar con un archivo `.env` para almacenar el `bearer token` en la variable `BEARER_TOKEN`**

```sh
BEARER_TOKEN='Bearer...'
```

### Importancíon de librerías e inicialización de variables
```python
import json
from pathlib import Path

import requests
from dotenv import dotenv_values, set_key

CONFIG = dotenv_values() # cargamos las variables existentes
env_file_path = Path(".env") # obtenemos el archivo .env
env_file_path.touch(mode=0o600, exist_ok=True) # otorgamos permisos de escritura

MANIFEST_URL = "https://apitest.dtes.mh.gob.sv/fesv/recepciondte" # para hacer peticiones de validaciones de DTE
BEARER_TOKEN_URL = "https://apitest.dtes.mh.gob.sv/seguridad/auth" # para obtener el refresh token (BEARER_TOKEN)

HEADERS = {
    "Authorization": "",
    "content-Type": "application/JSON",
    "User-Agent": "PostmanRuntime/7.32.3",
}

PAYLOAD = {
    "ambiente": "00",
    "idEnvio": 1,
    "version": 1,
    "tipoDte": "01",
    "documento": "",
    "codigoGeneracion": "",
}
```

### Declaramos la función para hacer las peticiones de validación de DTE's

```python
def send_request(manifest_url: str, signed: str, config: Dict) -> Dict:
    """ Parametros:
        manifest_url: url para validar DTE's
        signed: DTE firmado
        config: variables en archivo .env
    """
    HEADERS["Authorization"] = config.get("BEARER_TOKEN")
    PAYLOAD["documento"] = signed

    payload = f"{json.dumps(PAYLOAD)}"
    response = requests.post(
        manifest_url,
        headers=HEADERS,
        data=payload,
    )

    if response.status_code == 401:  # Si el status es 401 significa que el token caducó y debemos generar otro
        # llamamos a la función que se encarga de refrescar el token
        config = get_bearer_token(
            bearer_token_url=BEARER_TOKEN_URL, config=config
        )
        # Hacemos recursión para intentar validar el documento
        return send_request(
            manifest_url=manifest_url, signed=signed, config=config
        )

    # siginifica que nuestro token funciona nuevamente
    return json.loads(response.content)
```

### Declaramos la función para actualizar el token

```python
def get_bearer_token(bearer_token_url: str, config: Dict) -> Dict:
    """
        Parametros:
        bearer_token_url: para hacer la petición del refresh token
        config: variables en archivo .env
    """
    headers = {"Content-Type": "application/x-www-form-urlencoded"}
    # En el archivo .env se ingresan el usuario: (NIT DE LA EMPRESA)
    # y el password, generado desde el adminsitrador (dashboard) de DTE
    payload = f"user={config.get('NIT')}&pwd={config.get('API_PASS')}"
    response = requests.request(
        "POST", bearer_token_url, headers=headers, data=payload
    )

    assert response.status_code == 200 # si el estatus code no es 200 revisar credenciales
    data = json.loads(response.content) # cargamos la respuesta
    
    # guardamos el token en el archivo .env
    set_key(
        dotenv_path=env_file_path,
        key_to_set="BEARER_TOKEN",
        value_to_set=data.get("body").get("token"),
    )
    
    return dotenv_values()
```

**Resuesta que envía el generador del token (BEARER_TOKEN_URL)**
```python
{
    "status": "OK",
    "body": {
        "user": "TU USUARIO",
        "token": "Bearer TU TOKEN",
        "rol": {
            "nombre": "Usuario",
            "codigo": "ROLE_USER",
            "descripcion": null,
            "rolSuperior": null,
            "nivel": null,
            "activo": null,
            "permisos": null
        },
        "roles": [
            "ROLE_USER"
        ],
        "tokenType": "Bearer"
    }
}
```

**Importante:** este 'tutorial' solo pretende mostrar una posible solución (que me ha funcionado), por esa razón no continua con la explicación de la firma del documento o la generación de la estructura DTE.

No dude en contactarme a: gebisys@live.com si necesita mayor información sobre el tema.
