# Arquitectura y Flujo del Sistema — Password Generator & Vault

Documentación técnica del flujo de ejecución, validación de contraseñas y almacenamiento cifrado en `vault.enc`.

---

## 1. Diagrama de Flujo Principal (CLI)

```mermaid
flowchart TD
    Start([Inicio: main.py]) --> Banner[Mostrar Banner ASCII]
    Banner --> Confirm{¿Presionó Enter?}
    Confirm -- No --> Cancel([Inicio cancelado])
    Confirm -- Sí --> Menu[Menú Principal]

    Menu --> Op1[1. Generar contraseña]
    Menu --> Op2[2. Información y consejos]
    Menu --> Op3[3. Evaluar contraseña]
    Menu --> Op4[4. Password Vault]
    Menu --> Op5([5. Salir])

    %% Rama 1: Generador
    Op1 --> GenSub{Submenú Generador}
    GenSub -- 2. Regresar --> Menu
    GenSub -- 1. Generar --> GenPass[generar_password - 16 chars]
    GenPass --> AskSave{¿Guardar en Vault?}
    AskSave -- No --> GenSub
    AskSave -- Sí --> OpenV1[open_vault_interactively]
    OpenV1 --> SaveEntry[add_entry & lock]
    SaveEntry --> GenSub

    %% Rama 2: Consejos
    Op2 --> Tips[funciones.security_tips]
    Tips --> Menu

    %% Rama 3: Evaluador
    Op3 --> TestInput[getpass: Contraseña a evaluar]
    TestInput --> Eval[funciones.password_tester]
    Eval --> ShowRes[Mostrar validez o lista de problemas]
    ShowRes --> Again{¿Evaluar otra?}
    Again -- Sí --> TestInput
    Again -- No --> Menu

    %% Rama 4: Vault
    Op4 --> OpenV2[open_vault_interactively]
    OpenV2 -- Fallo / Cancelado --> Menu
    OpenV2 -- Desbloqueado --> VaultMenu{Menú Vault}
    VaultMenu -- 1. Listar --> ListE[list_entries] --> VaultMenu
    VaultMenu -- 2. Mostrar --> RevE[get_entry por ID] --> VaultMenu
    VaultMenu -- 3. Añadir --> AddE[add_entry manual] --> VaultMenu
    VaultMenu -- 4. Eliminar --> DelE[delete_entry por ID] --> VaultMenu
    VaultMenu -- 5. Bloquear --> LockV[vault.lock] --> Menu
```

---

## 2. Flujo de Autenticación y Cifrado (`PasswordVault`)

Detalle de cómo `VaultService` y `PasswordVault` gestionan la derivada de claves y la escritura atómica en disco:

```mermaid
flowchart LR
    subgraph Inicializacion [1. Acceso al Vault]
        Check{¿Existe vault.enc?}
        Check -- No --> Create[Pedir y confirmar Master Password >= 12 chars]
        Create --> SaltGen[Generar Salt aleatorio 16 bytes]
        Check -- Sí --> ReadEnv[Leer JSON Envelope de vault.enc]
        ReadEnv --> ValEnv[Validar versión, Scrypt y AES-256-GCM]
    end

    subgraph Criptografia [2. Derivación y Cifrado]
        SaltGen --> KDF[Scrypt KDF: N=2^15, r=8, p=1]
        ValEnv --> KDF
        KDF --> Key[Clave derivada 32 bytes]
        Key --> Nonce[Generar Nonce 12 bytes]
        Nonce --> AES[AES-256-GCM + Associated Data]
    end

    subgraph Persistencia [3. Escritura Segura]
        AES --> Tmp[Escribir en vault.enc.tmp + fsync]
        Tmp --> Replace[os.replace atómico a vault.enc]
        Replace --> Chmod[Permisos 600: Solo lectura/escritura propietario]
    end
```

---

## 3. Estructura de Módulos

| Archivo | Responsabilidad |
| :--- | :--- |
| `main.py` | Punto de entrada principal, menús interactivos y coordinación general. |
| `Password_Generator.py` | Generación criptográficamente segura (`secrets`) garantizando minúsculas, mayúsculas, dígitos y símbolos. |
| `funciones.py` | Reglas de validación de fortaleza (`password_tester`) y consejos de seguridad. |
| `vault_service.py` | Capa de servicio para validar coincidencia de contraseñas maestras (`compare_digest`) e instanciar el vault. |
| `Password_vault.py` | Motor criptográfico (`Scrypt` + `AESGCM`), gestión CRUD de credenciales y escritura atómica en disco. |
