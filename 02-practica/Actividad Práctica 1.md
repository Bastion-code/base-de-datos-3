# Practica 1 - Sistema de Gestión de Cafetería de Especialidad: "Caxambu NoSQL"

## 1. Definición del Dominio del Problema
El proyecto toma el diseño y la persistencia de datos para una plataforma de gestión en una cadena de cafeterías de especialidad y venta de café en grano. El negocio del café enfrenta el desafío de administrar un catálogo de productos altamente dinámico (variedades de café con múltiples atributos cambiantes como origen, nota de cata, altura de cultivo, proceso de secado y tipo de molienda) junto con un flujo masivo de pedidos en tiempo real provenientes tanto de las mesas físicas como de la tienda online.


Los sistemas relacionales tradicionales introducen una rigidez en los esquemas que dificulta la actualización o incorporación de nuevos atributos para los distintos orígenes del café, obligando además a realizar múltiples operaciones de combinación (*joins*) entre tablas de productos, categorías, pedidos y detalles para consolidar una sola transacción.

**Rol de la Base de Datos:**
    La base de datos documental (MongoDB) actuará como la infraestructura central de persistencia. Su esquema flexible e híbrido facilitará la convivencia de productos con propiedades sumamente heterogéneas (café en grano, pastelería, cafeteras, etc.) sin alterar la base de datos completa. Adicionalmente, el almacenamiento de pedidos estructurados en documentos únicos agilizará las operaciones de lectura y escritura en horas pico de consumo, garantizando un alto rendimiento y escalabilidad horizontal.

---

## 2. Esquema de las Colecciones (Mockup JSON)

### Colección: `clientes`
``` json 
{
  "_id": "usr_987654321",
  "nombre": "Sebastián Vidal",
  "email": "svidal050@gmail.com",
  "fecha_registro": "2026-08-18T16:02:14Z",
  "perfil": {
    "telefono": "+54911XXXXXXX",
    "preferencias": ["Café de Especialidad", "Filtrados", "Origen Colombia"]
  }
} 
```

### Colección: `productos`
``` json 
{
  "_id": "prod_cafe_001",
  "nombre": "Café Caxambu Colombiano",
  "categoria": "Café en Grano",
  "precio": 4500.00,
  "stock": 45,
  "detalles_especialidad": {
    "origen": "Colombia",
    "region": "Huila",
    "altura_msnm": 1750,
    "proceso": "Lavado Extended",
    "notas_catas": ["Frutos rojos", "Caramelo", "Acidez brillante"],
    "puntuacion_sca": 86.5
  }
} 
``` 

### Colección: `pedidos`
``` json 
{
  "_id": "ped_777888999",
  "cliente_id": "usr_987654321",
  "fecha_pedido": "2026-08-18T16:15:00Z",
  "tipo_entrega": "Take Away",
  "monto_total": 9000.00,
  "items": [
    {
      "producto_id": "prod_cafe_001",
      "nombre_producto": "Café Caxambu Colombiano",
      "cantidad": 2,
      "variante_molienda": "Fina (Espresso)",
      "precio_unitario": 4500.00
    }
  ],
  "estado": "Entregado"
}
```

## 3. Fundamentación de la Lógica No Relacional (Decisiones Arquitectónicas)

El modelado de las colecciones se estructuró analizando los patrones de acceso de la aplicación con el propósito de optimizar las consultas críticas y evitar operaciones costosas en el servidor:

### A) Uso de Documentos Anidados (Embedded Documents)

### Detalles de Especialidad en Productos:
    Los atributos específicos del café (origen, notas de cata, altura, etc.) se incluyeron de forma anidada dentro de cada documento de la colección productos. Dado que la información técnica de un grano de café no cambia de manera independiente a este y es de tamaño reducido, mantenerla integrada asegura que la aplicación renderice la ficha técnica del *producto* instantáneamente con una sola lectura física al disco duro.

### Detalle de Items en Pedidos: 
    Los artículos comprados se anidan en una lista dentro del pedido correspondiente. Al momento de facturar o consultar el historial de compras de una mesa, el sistema recupera el pedido con todos sus elementos en un único viaje a la base de datos (cero joins), asegurando un rendimiento veloz durante los picos de atención.

### B) Uso de Referencias (References)
### Relación Cliente y Pedido: 
    Se definió una estrategia de *referenciación* guardando únicamente el *cliente_id* dentro de los documentos de la colección *pedidos*. Los datos personales de un usuario (como su email o teléfono) son propensos a modificaciones. Si se duplicaran los datos del cliente dentro de cada pedido, se generaría redundancia lógica y problemas severos de consistencia. Al estar referenciados, cualquier actualización en la colección *clientes* se refleja de forma limpia sin impactar el histórico de ventas.

### Relación Producto y Pedido:
    En la lista de ítems de un pedido se copian solo los datos esenciales para la facturación histórica (nombre, cantidad y precio cobrado), pero manteniendo la referencia al *producto_id* original. Esto previene que los documentos de pedidos crezcan desmedidamente y nos permite realizar análisis comerciales cruzados vinculando el identificador del producto sin saturar la memoria caché del motor de base de datos.