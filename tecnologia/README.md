# Tecnología – Ejemplo de Clean Architecture

Esta carpeta guarda los recursos tecnológicos del proyecto, separados de la documentación (`docs/`).
Aquí se muestra **cómo se vería en código** el enfoque Clean Architecture para el flujo de
**pago en caja con esquela** (ADR-003 y ADR-004).

> No es una funcionalidad terminada: es un ejemplo para comprobar que el diseño es implementable.

## Stack propuesto

| Parte | Tecnología propuesta |
|---|---|
| Portal Web (computadora y celular) | React + TypeScript |
| Backend API | Node.js + NestJS (TypeScript) |
| Base de datos | PostgreSQL |
| Caché | Redis |
| Cola de tareas | Cola de mensajes + workers en Node.js |
| Contenedores y despliegue | Docker en la nube o en el centro de datos municipal |
| Control de versiones | Git y GitHub |

## Estructura del ejemplo

```
src/modulos/pagos/
├── dominio/           Esquela.ts, PagoRegistrado.ts, contratos.ts
├── aplicacion/        GenerarEsquela.ts, RegistrarPagoEnCaja.ts
├── infraestructura/   adaptadoresEnMemoria.ts
└── composicion.ts     conecta contratos con adaptadores
```

## 1. Dominio (no importa nada externo)

```typescript
// dominio/Esquela.ts
export interface DatosEsquela {           // forma en que la esquela se guarda y se lee
  codigo: string;
  numeroExpediente: string;
  monto: number;                         // en céntimos: 15950 = S/ 159.50
  numeroRecibo: string | null;
}

export class Esquela {
  private constructor(
    readonly codigo: string,
    readonly numeroExpediente: string,
    readonly monto: number,               // en céntimos y desde el catálogo; nunca se escribe a mano
    private numeroRecibo: string | null,
  ) {}

  static nueva(codigo: string, numeroExpediente: string, monto: number): Esquela {
    if (!Number.isInteger(monto) || monto <= 0) throw new Error('El monto debe ser un número entero de céntimos mayor que cero');
    return new Esquela(codigo, numeroExpediente, monto, null);
  }

  static reconstruir(d: DatosEsquela): Esquela {        // al leerla de la base, conserva si ya fue pagada
    return new Esquela(d.codigo, d.numeroExpediente, d.monto, d.numeroRecibo);
  }

  static generarCodigo(numeroExpediente: string): string {
    return `ESQ-${numeroExpediente}`;      // una sola esquela por expediente, ej.: ESQ-EXP-2026-0001
  }

  registrarPago(numeroRecibo: string): void {
    if (!numeroRecibo.trim()) throw new Error('Falta el número de recibo');
    if (this.numeroRecibo) throw new Error('Esta esquela ya fue pagada');   // regla: se paga una sola vez
    this.numeroRecibo = numeroRecibo;
  }

  estaPagada(): boolean { return this.numeroRecibo !== null; }

  aDatos(): DatosEsquela {
    return { codigo: this.codigo, numeroExpediente: this.numeroExpediente, monto: this.monto, numeroRecibo: this.numeroRecibo };
  }
}

// dominio/PagoRegistrado.ts  (evento de dominio)
export interface PagoRegistrado {
  codigoEsquela: string;
  numeroExpediente: string;
  numeroRecibo: string;
}

// dominio/contratos.ts  (puertos)
import { Esquela } from './Esquela';
import { PagoRegistrado } from './PagoRegistrado';

export interface RepositorioEsquelas {
  buscarPorCodigo(codigo: string): Promise<Esquela | null>;
  buscarPorExpediente(numeroExpediente: string): Promise<Esquela | null>;
  guardarNueva(esquela: Esquela): Promise<boolean>; // ATÓMICO: false si el expediente ya tiene esquela
  guardarPago(esquela: Esquela): Promise<boolean>;  // ATÓMICO: false si en la base ya estaba pagada
}

export interface ConsultaTasa {            // la tasa vive solo en el catálogo (ADR-002)
  tasaDe(codigoReceta: string): Promise<number>;   // en céntimos
}

export interface GeneradorDocumentos {
  generarPdfEsquela(esquela: Esquela): Promise<void>;
}

export interface PublicadorEventos {
  publicar(evento: PagoRegistrado): Promise<void>;
}
```

## 2. Aplicación (casos de uso)

```typescript
// aplicacion/GenerarEsquela.ts
import { Esquela } from '../dominio/Esquela';
import { RepositorioEsquelas, ConsultaTasa, GeneradorDocumentos } from '../dominio/contratos';

export class GenerarEsquela {
  constructor(                                        // DIP: depende solo de contratos
    private readonly esquelas: RepositorioEsquelas,
    private readonly catalogo: ConsultaTasa,
    private readonly documentos: GeneradorDocumentos,
  ) {}

  // Se llama solo cuando el nivel de riesgo ya está confirmado (ADR-004)
  async ejecutar(numeroExpediente: string, codigoReceta: string): Promise<Esquela> {
    const existente = await this.esquelas.buscarPorExpediente(numeroExpediente);
    if (existente) {                                                         // una sola esquela por expediente
      if (!existente.estaPagada()) await this.documentos.generarPdfEsquela(existente);   // vuelve a armar el PDF (p. ej., se perdió)
      return existente;
    }
    const monto = await this.catalogo.tasaDe(codigoReceta);                  // RF05: tasa del TUPA
    const esquela = Esquela.nueva(Esquela.generarCodigo(numeroExpediente), numeroExpediente, monto);
    const creada = await this.esquelas.guardarNueva(esquela);
    if (!creada) {                                                           // otra petición la creó al mismo tiempo
      const otra = await this.esquelas.buscarPorExpediente(numeroExpediente);
      if (!otra) throw new Error('No se pudo leer la esquela del expediente');
      return otra;
    }
    await this.documentos.generarPdfEsquela(esquela);                        // el worker arma el PDF
    return esquela;
  }
}

// aplicacion/RegistrarPagoEnCaja.ts
import { RepositorioEsquelas, PublicadorEventos } from '../dominio/contratos';

export class RegistrarPagoEnCaja {
  constructor(
    private readonly esquelas: RepositorioEsquelas,
    private readonly eventos: PublicadorEventos,
  ) {}

  async ejecutar(codigo: string, numeroRecibo: string): Promise<void> {
    const esquela = await this.esquelas.buscarPorCodigo(codigo);
    if (!esquela) throw new Error('Esquela no encontrada');
    esquela.registrarPago(numeroRecibo);                         // 1.ª defensa: regla del dominio
    const guardado = await this.esquelas.guardarPago(esquela);   // 2.ª defensa: la base de datos
    if (!guardado) throw new Error('Esta esquela ya fue pagada');   // otro cajero la pagó al mismo tiempo
    // Ejemplo simple: publica directo. En producción se usaría outbox (ver la nota de la sección 3)
    await this.eventos.publicar({                                // Trámites reacciona y avanza el expediente
      codigoEsquela: esquela.codigo,
      numeroExpediente: esquela.numeroExpediente,
      numeroRecibo,
    });
  }
}
```

## 3. Infraestructura (adaptadores intercambiables)

El adaptador en memoria guarda **copias de los datos** (no el objeto), igual que una base de datos. Así el ejemplo
se comporta como PostgreSQL: cada lectura crea un objeto nuevo con `Esquela.reconstruir()`.

```typescript
// infraestructura/adaptadoresEnMemoria.ts
import { Esquela, DatosEsquela } from '../dominio/Esquela';
import { PagoRegistrado } from '../dominio/PagoRegistrado';
import { RepositorioEsquelas, ConsultaTasa, GeneradorDocumentos, PublicadorEventos } from '../dominio/contratos';

export class RepositorioEsquelasEnMemoria implements RepositorioEsquelas {
  private filas = new Map<string, DatosEsquela>();      // clave: código de la esquela

  async buscarPorCodigo(codigo: string) {
    const fila = this.filas.get(codigo);
    return fila ? Esquela.reconstruir({ ...fila }) : null;
  }
  async buscarPorExpediente(numero: string) {
    const fila = [...this.filas.values()].find(f => f.numeroExpediente === numero);
    return fila ? Esquela.reconstruir({ ...fila }) : null;
  }
  async guardarNueva(e: Esquela) {
    const yaExiste = [...this.filas.values()].some(f => f.numeroExpediente === e.numeroExpediente);
    if (yaExiste) return false;                                  // como UNIQUE (numero_expediente)
    this.filas.set(e.codigo, e.aDatos());
    return true;
  }
  async guardarPago(e: Esquela) {
    const actual = this.filas.get(e.codigo);
    if (!actual || actual.numeroRecibo !== null) return false;   // ya estaba pagada
    this.filas.set(e.codigo, e.aDatos());
    return true;
  }
}

export class ConsultaTasaEnMemoria implements ConsultaTasa {
  private tasas = new Map<string, number>([['LIC-01', 15950], ['LIC-03', 30590]]);   // TUPA 2024, en céntimos
  async tasaDe(codigoReceta: string) {
    const tasa = this.tasas.get(codigoReceta);
    if (tasa === undefined) throw new Error('Receta sin tasa en el catálogo');
    return tasa;
  }
}

export class GeneradorDocumentosFalso implements GeneradorDocumentos {
  async generarPdfEsquela(e: Esquela) { console.log(`PDF encolado para ${e.codigo}`); }
}

export class PublicadorEventosEnMemoria implements PublicadorEventos {
  publicados: PagoRegistrado[] = [];
  async publicar(evento: PagoRegistrado) { this.publicados.push(evento); }
}
```

### Cómo lo cumple PostgreSQL (`RepositorioEsquelasPostgres`)

```sql
-- guardarNueva: atómico, un expediente nunca tiene dos esquelas (UNIQUE sobre numero_expediente)
INSERT INTO esquelas (codigo, numero_expediente, monto_centimos) VALUES ($1, $2, $3)   -- monto_centimos INTEGER
ON CONFLICT (numero_expediente) DO NOTHING;
-- 1 fila insertada = esquela creada (true); 0 filas = el expediente ya tenía esquela (false)

-- guardarPago: solo actualiza si la esquela sigue sin pagar
UPDATE esquelas SET numero_recibo = $2, pagada_en = now()
WHERE codigo = $1 AND numero_recibo IS NULL;
-- 1 fila actualizada = pago registrado (true); 0 filas = ya estaba pagada (false)
```

### Nota para producción: outbox (no forma parte de este ejemplo)

En el ejemplo, `RegistrarPagoEnCaja` guarda el pago y **después** llama a `publicar()`. Si la publicación fallara,
el pago quedaría registrado pero el expediente no avanzaría. En producción se evitaría con un **outbox**:

1. El pago y el evento se guardan **en una sola sentencia**, y el evento **solo** se guarda si el pago se registró.
2. Los **workers** leen la tabla `eventos_pendientes` y entregan el evento a Trámites; si falla, lo reintentan.
3. En ese diseño, el caso de uso **ya no llama a `publicar()`**, así el evento no se entrega dos veces.
4. En la **misma transacción** se guarda también la fila de auditoría del pago (RF25); se omite abajo para que la sentencia se lea fácil.
5. Un evento puede entregarse **más de una vez** (por ejemplo, si el worker cae justo después de procesarlo). Por eso Trámites es
   **idempotente**: guarda el id del evento en `eventos_procesados` en la misma transacción en que avanza el expediente y, si el id
   ya existe, no hace nada. Así un reintento nunca emite dos licencias. Varios workers no toman el mismo evento gracias a
   `FOR UPDATE SKIP LOCKED`.

```sql
-- Si el UPDATE no actualiza ninguna fila (ya estaba pagada), el INSERT no guarda nada
WITH pago AS (
  UPDATE esquelas SET numero_recibo = $2, pagada_en = now()
  WHERE codigo = $1 AND numero_recibo IS NULL
  RETURNING codigo, numero_expediente, numero_recibo
)
INSERT INTO eventos_pendientes (id, tipo, datos)
SELECT gen_random_uuid(), 'PagoRegistrado',
       json_build_object('codigoEsquela', codigo, 'numeroExpediente', numero_expediente, 'numeroRecibo', numero_recibo)
FROM pago;
```

```sql
-- Worker: toma eventos que ningún otro worker está procesando
SELECT id, tipo, datos FROM eventos_pendientes
WHERE entregado_en IS NULL
ORDER BY creado_en
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- Trámites, en la misma transacción en que avanza el expediente
INSERT INTO eventos_procesados (id_evento) VALUES ($1)
ON CONFLICT (id_evento) DO NOTHING;
-- 0 filas insertadas = evento repetido: no se avanza el expediente ni se emite otra licencia
```

## 4. Composición (único lugar que elige los adaptadores)

```typescript
// composicion.ts
import { GenerarEsquela } from './aplicacion/GenerarEsquela';
import { RegistrarPagoEnCaja } from './aplicacion/RegistrarPagoEnCaja';
import {
  RepositorioEsquelasEnMemoria, ConsultaTasaEnMemoria,
  GeneradorDocumentosFalso, PublicadorEventosEnMemoria,
} from './infraestructura/adaptadoresEnMemoria';

const repo = new RepositorioEsquelasEnMemoria();        // en producción: RepositorioEsquelasPostgres
const eventos = new PublicadorEventosEnMemoria();
const generar = new GenerarEsquela(repo, new ConsultaTasaEnMemoria(), new GeneradorDocumentosFalso());
const registrar = new RegistrarPagoEnCaja(repo, eventos);

const esquela = await generar.ejecutar('EXP-2026-0001', 'LIC-01');   // riesgo bajo: S/ 159.50 del catálogo
console.log(esquela.codigo, esquela.monto);                          // ESQ-EXP-2026-0001 15950 (S/ 159.50)

const otraVez = await generar.ejecutar('EXP-2026-0001', 'LIC-01');   // se pide de nuevo (p. ej., se perdió el PDF)
console.log(otraVez.codigo === esquela.codigo);                      // true: la misma esquela, nunca una segunda (su PDF se vuelve a encolar)

// Dos cajeros registran la MISMA esquela al mismo tiempo
const resultados = await Promise.allSettled([
  registrar.ejecutar(esquela.codigo, 'REC-000123'),
  registrar.ejecutar(esquela.codigo, 'REC-000124'),
]);
console.log(resultados.map(r => r.status));        // [ 'fulfilled', 'rejected' ] → solo un pago se registra
console.log(eventos.publicados.length);            // 1

const leida = await repo.buscarPorCodigo(esquela.codigo);
console.log(leida?.estaPagada());                  // true: al leerla de nuevo sigue pagada
```

## Qué demuestra este ejemplo

| Idea | Dónde se ve |
|---|---|
| El dominio no depende de la tecnología | `Esquela` y los contratos no importan nada externo. |
| Una sola fuente de la tasa | `GenerarEsquela` pide el monto a `ConsultaTasa`; nadie lo escribe a mano. |
| La esquela pagada sigue pagada al leerla | `Esquela.reconstruir()` recibe el número de recibo guardado. |
| Un solo pago aunque dos cajeros actúen a la vez | `guardarPago` es atómico: en PostgreSQL, `UPDATE ... WHERE numero_recibo IS NULL`. |
| Una sola esquela por expediente | `GenerarEsquela` devuelve la existente; `guardarNueva` es atómico (en PostgreSQL, `ON CONFLICT (numero_expediente) DO NOTHING`), así nadie paga dos veces. |
| Inversión de dependencias (DIP) | Los casos de uso reciben sus contratos por el constructor. |
| Adaptadores intercambiables (LSP) | El contrato exige que `guardarNueva` y `guardarPago` sean atómicos; el adaptador en memoria y el de PostgreSQL lo cumplen, por eso se cambian sin tocar los casos de uso. |
| Límite entre módulos | Pagos no toca el `Expediente`: publica `PagoRegistrado` y Trámites reacciona. |
| Cómo evitar perder el evento | El ejemplo publica directo para ser simple; en producción se usaría outbox (nota de la sección 3). |
| Dinero sin errores de redondeo | Los montos se manejan en céntimos enteros (`15950` = S/ 159.50); en PostgreSQL, columna `INTEGER`. |
| Esquela recuperable | Si se vuelve a pedir una esquela no pagada, `GenerarEsquela` encola otra vez su PDF. |
| Eventos sin efectos duplicados | En producción, Trámites procesa `PagoRegistrado` de forma idempotente (`eventos_procesados`, sección 3). |
| Se prueba sin base de datos | Todo el ejemplo funciona en memoria. |

**Fuera de este ejemplo** (están en el diseño interno de módulos): la pantalla de confirmación del cajero
(`ConsultarEsquela`), la corrección del número de recibo por la jefatura (`CorregirNumeroRecibo`) y el registro de auditoría.