<template>
  <div class="container">
    <div class="superior">
      <div class="titulo">Subir Archivo CSV</div>
      <div>Subir Archivo CSV:</div>
      <div>
        <div class="seleccionar">
          <input type="file" accept=".csv" @change="procesarArchivo" style="display: none;" ref="fileInput">
          <button @click="abrirSeleccionArchivo" class="btn"
            style="background-color: #007BFF; font-weight: bold; color: white; width: 25%; height: 30%;">
            Seleccionar el archivo
          </button>
          <div v-if="nombreArchivo">{{ nombreArchivo }}</div>
          <div v-else>Ningún archivo seleccionado</div>
        </div>
      </div>

      <!-- Botones de acción -->
      <div class="btn-agregar">
        <button @click="enviarFacturasAnidadas" class="btn" style="color: white; width: 18%;">Enviar Facturas Anidadas a
          Alegra</button>
        <button @click="enviarFacturasSimples" class="btn" style="color: white; width: 18%;">Enviar Facturas Simples a
          Alegra</button>
        <button @click="procesarDatos" class="btn" style="color: white; width: 18%;">Procesar Datos</button>
        <button @click="limpiarBaseDeDatos" class="btn" style="color: white; width: 18%;">Limpiar Base de Datos</button>
        <button @click="subirArchivo" class="btn" style="color: white; width: 18%;">Subir</button>
      </div>

      <!-- Tabla -->
      <div style="width: 90vw;">
        <q-table class="my-sticky-virtscroll-table" virtual-scroll flat bordered v-model:pagination="pagination"
          :rows-per-page-options="[10, 30, 50]" :virtual-scroll-sticky-size-start="48" row-key="name" :rows="rows"
          :columns="columns">
          <!-- Columna para el estado -->
          <template v-slot:body-cell-estado="props">
            <q-td :props="props">
              <label v-if="props.row.estado == 1" style="color: green">Activo</label>
              <label v-else style="color: red">Inactivo</label>
            </q-td>
          </template>

          <!-- Columna para las opciones de editar/activar/inactivar -->
          <template v-slot:body-cell-opciones="props">
            <q-td :props="props" class="botones">
              <button @click="imprimirTicket(props.row)" class="edi">
                <i class="fa-solid fa-print"></i>
              </button>
            </q-td>
          </template>
        </q-table>
      </div>
    </div>
  </div>
</template>

<script>
import { jsPDF } from 'jspdf'; // Importar jsPDFx
import axios from 'axios';
import fondo from "../assets/fondo.png"
import logo from "../assets/logo.png"
import cufe from "../assets/cufe.png"
import cliente from "../assets/cliente.png"
import pedido from "../assets/tabla-de-pedido.png"
import escanear from "../assets/escanear.png"

export default {
  data() {
    return {
      nombreArchivo: '', // Para mostrar el nombre del archivo seleccionado
      rows: [], // Filas de la tabla
      columns: [ // Columnas de la tabla
        { name: 'nombre', label: 'Nombre', align: 'left', field: row => row.nombre },
        { name: 'identificacion', label: 'C.C', align: 'left', field: 'identificacion' },
        { name: 'codigo_producto', label: 'Código', align: 'left', field: 'codigo_producto' },
        { name: 'fecha_vencimiento ', label: 'Fecha Vencimiento', align: 'left', field: 'fecha_vencimiento' },
        { name: 'fecha_elaboracion ', label: 'Fecha Elaboracion', align: 'left', field: 'fecha_elaboracion' },
        { name: 'anidar ', label: 'Anidar', align: 'left', field: 'anidar' },

        {
          name: "opciones",
          label: "Opciones",
          field: (row) => null,
          sortable: false,
          align: "center",
        }
      ],

      pagination: {
        page: 1,
        rowsPerPage: 10
      },
    };
  },
  methods: {
    abrirSeleccionArchivo() {
      this.$refs.fileInput.click();
    },
    procesarArchivo(event) {
      const archivo = event.target.files[0];
      if (archivo) {
        this.nombreArchivo = archivo.name;
      } else {
        this.nombreArchivo = '';
      }
    },

    calcularTotal(cantidadProducto, valor_unitario, codigoImpuestoCargo, reteica, retencion) {
      // Asegurarte de que todos los valores son números válidos
      cantidadProducto = isNaN(cantidadProducto) ? 0 : parseFloat(cantidadProducto);
      valor_unitario = isNaN(valor_unitario) ? 0 : parseFloat(valor_unitario);
      codigoImpuestoCargo = isNaN(codigoImpuestoCargo) ? 0 : parseFloat(codigoImpuestoCargo);
      reteica = isNaN(reteica) ? 0 : parseFloat(reteica);
      retencion = isNaN(retencion) ? 0 : parseFloat(retencion);

      const valorBruto = cantidadProducto * valor_unitario;
      const descuento = (valorBruto * codigoImpuestoCargo) / 100;
      const total = valorBruto + descuento - reteica - retencion;
      return total.toFixed(0);  // Retornar con dos decimales
    },

    calcularSubtotal(cantidadProducto, valor_unitario) {
      // Asegurarte de que ambos valores sean números válidos
      cantidadProducto = isNaN(cantidadProducto) ? 0 : parseFloat(cantidadProducto);
      valor_unitario = isNaN(valor_unitario) ? 0 : parseFloat(valor_unitario);

      const subtotal = cantidadProducto * valor_unitario;
      return subtotal.toFixed(0);  // Retornar sin decimales
    },
    async subirArchivo() {
      const archivo = this.$refs.fileInput.files[0];
      if (!archivo) {
        alert('Por favor, selecciona un archivo CSV');
        return;
      }

      const reader = new FileReader();

      reader.onload = (event) => {
        const contenido = event.target.result;
        const filas = contenido.split('\n').map(fila => fila.split(';'));

        this.rows = filas.slice(1).map((fila, index) => {
          const cantidadProducto = parseFloat(fila[3]) || 0;
          const valor_unitario = parseFloat(fila[4]) || 0;

          return {
            // Otros campos de la fila...
            subtotal: this.calcularSubtotal(cantidadProducto, valor_unitario),
          };
        });

        // Ignorar el encabezado y convertir las filas en un objeto para usar en la tabla
        this.rows = filas.slice(1).map((fila, index) => {
          const cantidadProducto = parseFloat(fila[3]) || 0;
          const valor_unitario = parseFloat(fila[4]) || 0;
          const codigoImpuestoCargo = parseFloat(fila[5 ]) || 0;
          const reteica = parseFloat(fila[8]) || 0;
          const retencion = parseFloat(fila[6]) || 0;

          return {
            nombre: fila[0],
            identificacion: fila[1],
            codigo_producto: fila[2],
            cantidadProducto: cantidadProducto,
            valor_unitario: valor_unitario,
            codigoImpuestoCargo: fila[5],
            retencion: retencion,
            valorRetencion: fila[7],
            reteica: reteica,
            valorReteica: fila[9],
            codigoImpuestoCargo: codigoImpuestoCargo,
            fecha_vencimiento: fila[11],
            fecha_elaboracion: fila[12],
            anidar: fila[13] || '',
            total: this.calcularTotal(cantidadProducto, valor_unitario, codigoImpuestoCargo, reteica, retencion),
          };
        });
      }

      reader.onerror = (error) => {
        console.error('Error al leer el archivo:', error);
      };

      reader.readAsText(archivo);
    },

    enviarFacturasAnidadas() {
      // Lógica para enviar facturas anidadas
    },
    enviarFacturasSimples() {
      // Lógica para enviar facturas simples
    },
    procesarDatos() {
      // Lógica para procesar los datos
    },
    limpiarBaseDeDatos() {
      // Lógica para limpiar la base de datos
    },
    imprimirTicket(factura) {
      const doc = new jsPDF({
        orientation: "portrait",
        unit: "mm",
        format: [140, 216],
      })

      // Dimensiones y colores
      const headerHeight = 18;
      const headerWidth = 136;
      const pageWidth = 140;
      const pageHeight = 216;
      const cornerRadius = 5;

      // Añadir las plantillas
      doc.addImage(fondo, "PNG", 0, 0, pageWidth, pageHeight);
      doc.addImage(logo, "PNG", 2, 1, 50, 25);
      doc.addImage(cliente, "PNG", 1, 60, 138, 24);
      doc.addImage(pedido, "PNG", 1, 104, 136, 56);
      doc.addImage(escanear, "PNG", -1, 162, 142, 32);

      // Título "FACTURA ELECTRÓNICA" en negrita
      const title3 = "FACTURA ELECTRÓNICA";
      const titleFontSize3 = 10;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(titleFontSize3);
      doc.setTextColor(0, 0, 0);

      const title3Width = (doc.getStringUnitWidth(title3) * titleFontSize3) / doc.internal.scaleFactor;
      const title3X = headerWidth - title3Width - 2;
      const title3Y = headerHeight / 2 + titleFontSize3 / 2 - 7;
      doc.text(title3, title3X, title3Y);

      // Añadir la imagen PQR CUFE
      const pqrX = 65; // Posición horizontal (ajusta según sea necesario)
      const pqrY = 3; // 2 cm desde la parte superior
      const pqrWidth = 18; // Ancho de la imagen
      const pqrHeight = 16; // Altura de la imagen
      doc.addImage(cufe, "JPEG", pqrX, pqrY, pqrWidth, pqrHeight);

      // Texto "CUFE:" en negrita, centrado debajo de la imagen PQR
      const cufeTitle = "CUFE:";
      const cufeFontSize = 12;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(cufeFontSize);
      const cufeWidth = (doc.getStringUnitWidth(cufeTitle) * cufeFontSize) / doc.internal.scaleFactor;
      const cufeTitleX = pqrX + (pqrWidth - cufeWidth) / 2;
      const cufeTitleY = pqrY + pqrHeight + 5; // Ajustar la distancia debajo de la imagen PQR
      doc.text(cufeTitle, cufeTitleX, cufeTitleY);

      // Dibujar una barra de progreso de 3 puntos en la mitad de la hoja
      const progressBarY = pageHeight / 6; // Centrado verticalmente
      const progressBarXStart = (pageWidth - 90) / 2; // Ajustar para centrar horizontalmente
      const circleRadius = 2.5;
      const circleSpacing = 45; // distancia
      doc.setDrawColor(226, 48, 9); // Color del borde de los círculos
      doc.setFillColor(255, 255, 255); // Color de relleno de los círculos (blanco)

      for (let i = 0; i < 3; i++) {
        const circleX = progressBarXStart + i * circleSpacing;
        doc.circle(circleX, progressBarY, circleRadius, "FD"); // 'FD' para borde y relleno
        if (i < 2) {
          const nextCircleX = circleX + circleSpacing;
          doc.setLineWidth(0.5); // Reducir el grosor de la línea
          doc.line(
            circleX + circleRadius + 0.5,
            progressBarY,
            nextCircleX - circleRadius - 0.5,
            progressBarY
          ); // Ajustar para evitar superposición
        }
      }

      // Añadir las fechas sobre los puntos
      const fechas = ["25/08", "30/08", "05/09"];
      const fechaFontSize = 8;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(fechaFontSize);
      for (let i = 0; i < fechas.length; i++) {
        const fechaX = progressBarXStart + i * circleSpacing;
        const fechaY = progressBarY - 5; // Ajustar la posición vertical para colocar el texto sobre los puntos
        doc.text(
          fechas[i],
          fechaX -
          (doc.getStringUnitWidth(fechas[i]) * fechaFontSize) /
          (2 * doc.internal.scaleFactor),
          fechaY
        );
      }

      // Añadir el contenido de tiempo debajo de los puntos
      const tiempo = [
        "Inicio de período",
        "Fecha de suspensión",
        "Fin de período",
      ];

      const tiempoFontSize = 8;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(tiempoFontSize);
      for (let i = 0; i < tiempo.length; i++) {
        const tiempoX = progressBarXStart + i * circleSpacing;
        const tiempoY = progressBarY + 7; // Ajustar la posición vertical para colocar el texto debajo de los puntos
        doc.text(
          tiempo[i],
          tiempoX -
          (doc.getStringUnitWidth(tiempo[i]) * tiempoFontSize) /
          (2 * doc.internal.scaleFactor),
          tiempoY
        );
      }

      // Cuadro que ocupa todo el ancho con el texto "Hola Mundo" centrado
      const cuadroX = 0;
      const cuadroYy = progressBarY + 10;
      const cuadroWidtht = pageWidth;
      const cuadroHeightt = 10;

      doc.setDrawColor(0, 0, 0); // Color del borde
      doc.setFillColor(255, 255, 255); // Color de relleno
      doc.rect(cuadroX, cuadroYy, cuadroWidtht, cuadroHeightt, "FD"); // Dibuja el cuadro

      // Texto "DETALLE DEL CLIENTE" centrado dentro del cuadro
      const holaMundo = "DETALLE DEL CLIENTE";
      const holaFontSize = 15;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(holaFontSize);
      const holaWidth = (doc.getStringUnitWidth(holaMundo) * holaFontSize) / doc.internal.scaleFactor;
      const holaX = (pageWidth - holaWidth) / 2;
      const holaY = cuadroYy + cuadroHeightt / 5 + holaFontSize / 3; // Centrar verticalmente el texto
      doc.text(holaMundo, holaX, holaY);

      // Cuadro que ocupa todo el ancho con el texto centrado
      const cuadrox = 0;
      const cuadroy = progressBarY + 55;
      const cuadrowidtht = pageWidth;
      const cuadroheightt = 10;
      doc.setDrawColor(0, 0, 0); // Color del borde
      doc.setFillColor(255, 255, 255); // Color de relleno
      doc.rect(cuadrox, cuadroy, cuadrowidtht, cuadroheightt, "FD"); // Dibuja el cuadro

      // Texto "DETALLE DE LA COMPRA" centrado dentro del cuadro
      const compra = "DETALLE DE LA COMPRA";
      const compraFontSize = 15;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(compraFontSize);
      const compraWidth = (doc.getStringUnitWidth(compra) * compraFontSize) / doc.internal.scaleFactor;
      const compraX = (pageWidth - compraWidth) / 2;
      const compraY = cuadroYy + cuadroHeightt + 42; // Centrar verticalmente el texto
      doc.text(compra, compraX, compraY);

      // Dimensiones y posición del cuadro
      const cuadroY = pageHeight - 20.3; // Posición vertical del cuadro (ajustar según sea necesario)
      const cuadroHeight = 20; // Altura del cuadro
      const cuadroWidth = pageWidth; // Ancho del cuadro
      const cellHeightt = cuadroHeight / 2; // Altura de cada fila
      const cellWidtht = cuadroWidth / 2; // Ancho de cada columna

      // Dibujar el cuadro
      doc.setDrawColor(0, 0, 0); // Color del borde
      doc.setFillColor(255, 255, 255); // Color de relleno
      doc.rect(0, cuadroY, cuadroWidth, cuadroHeight, "FD"); // Dibuja el cuadro con borde y relleno

      // Textos para cada celda (2 filas y 2 columnas)
      const texto = [
        ["NOMBRE:", "VALOR TOTAL: "],
        ["CEDULA:", "MES:"],
      ];

      for (let i = 0; i < texto.length; i++) {
        for (let j = 0; j < texto[i].length; j++) {
          const text = texto[i][j];
          const textFontSize = 9;
          doc.setFont("Helvetica", "bold");
          doc.setFontSize(textFontSize);

          // Posicionar el texto en cada celda
          const textY =
            cuadroY + i * cellHeightt + cellHeightt / 10 + textFontSize / 2;
          let textX;

          // Alinear texto a la izquierda
          textX = j * cellWidtht + 2; // Ajuste de margen izquierdo

          doc.text(text, textX, textY);

          // Dibujar las líneas de división horizontal y vertical
          if (i < texto.length - 1) {
            doc.line(
              0,
              cuadroY + cellHeightt * (i + 1),
              cuadroWidth,
              cuadroY + cellHeightt * (i + 1)
            ); // Línea horizontal
          }
          if (j < texto[i].length - 1) {
            doc.line(
              cellWidtht * (j + 1),
              cuadroY,
              cellWidtht * (j + 1),
              cuadroY + cuadroHeight
            ); // Línea vertical
          }
        }
      };

      // Texto "OBSERVACIONES" centrado dentro del cuadro
      const observaciones = "OBSERVACIONES";
      const observacionfontzise = 10;
      doc.setFont("Helvetica", "bold");
      doc.setFontSize(observacionfontzise);
      const observacionWidth =
        (doc.getStringUnitWidth(observaciones) * observacionfontzise) /
        doc.internal.scaleFactor;
      let observacionx = (pageWidth - observacionWidth) / 2;
      let observaciony =
        cuadroYy + cuadroHeightt / 1 + observacionfontzise / 3; // Centrar verticalmente el texto
      observacionx -= 52; // Mover 10 unidades a la izquierda
      observaciony += 77; // Mover 20 unidades hacia abajo
      doc.text(observaciones, observacionx, observaciony);

      // Añadir los datos
      doc.setFont("Helvetica", "normal");
      doc.setFontSize(8);
      const offsetX = 90; // Ajustar el inicio del texto a la derecha
      const offsetY = cuadroYy + 14; // Ajustar el inicio del texto

      factura.total = this.calcularTotal(
        factura.cantidadProducto,
        factura.valor_unitario,
        factura.codigoImpuestoCargo,
        factura.reteica,
        factura.retencion
      );
      
      factura.subtotal= this.calcularSubtotal(
        factura.cantidadProducto,
        factura.valor_unitario
      )
      doc.text(` ${factura.identificacion}`, offsetX - 55, offsetY + 13);
      doc.text(` ${factura.direccion}`, offsetX + 15, offsetY + 13);
      doc.text(` ${factura.correo}`, offsetX - 65, offsetY + 21);
      doc.text(` ${factura.telefono}`, offsetX + 15, offsetY + 21);
      doc.text(` ${factura.codigo_producto}`, offsetX - 75, offsetY + 53);
      doc.text(` ${factura.cantidadProducto}`, offsetX - 43, offsetY + 53);
      doc.text(` ${factura.valor_unitario}`, offsetX - 29, offsetY + 53);
      doc.text(` ${factura.codigoImpuestoCargo}`, offsetX - 15, offsetY + 53);
      doc.text(` ${factura.retencion}`, offsetX - 4.5, offsetY + 53);
      doc.text(` ${factura.reteica}`, offsetX + 10, offsetY + 53);
      // doc.text(` ${factura.total}`, offsetX + 29, offsetY + 53); //total
      // doc.text(` ${factura.total}`, offsetX + 29, offsetY + 53); //subtotal 
      doc.text(` ${factura.total}`, offsetX + 29, offsetY + 97); //total inferior 
      doc.text(` ${factura.valor_unitario}`, offsetX + 29, offsetY + 75);
      doc.text(` ${factura.codigoImpuestoCargo}`, offsetX + 30, offsetY + 80); //
      doc.text(` ${factura.retencion}`, offsetX + 30, offsetY + 86); //
      doc.text(` ${factura.reteica}`, offsetX + 30, offsetY + 92); //
      doc.text(` ${factura.identificacion}`, offsetX - 55, offsetY + 151); // tirilla
      doc.text(` 27/09/2024`, offsetX + 12, offsetY + 151); // tirilla

      const subtotal= Number(factura.subtotal).toLocaleString('es-Es');
      doc.text(`${subtotal}`,offsetX + 29, offsetY + 53);
      // Formatear el total para que muestre los puntos de mil
      const totalFormateado = Number(factura.total).toLocaleString('es-ES');
      doc.text(`$${totalFormateado}`, offsetX + 15, offsetY + 141); // tirilla

      function limitarCaracteres(texto, limiteCaracteres) {
        if (texto.length > limiteCaracteres) {
          return texto.slice(0, limiteCaracteres) + '...';
        }
        return texto;
      }

      const nombredetalleclien = limitarCaracteres(factura.nombre, 63);
      doc.text(` ${nombredetalleclien}`, offsetX - 64, offsetY + 5);

      const nombreLimitado = limitarCaracteres(factura.nombre, 25);
      doc.text(` ${nombreLimitado}`, offsetX - 70, offsetY + 141); // tirilla 



      // doc.save('ticket.pdf');
      const pdfBlob = doc.output('blob');
      const pdfUrl = URL.createObjectURL(pdfBlob);
      window.open(pdfUrl, '_blank');  
      const link = document.createElement('a');
link.href = pdfUrl;
link.download = `Factura Electronica ${factura.nombre}.pdf`; // Nombre del archivo
link.click();
    }, 
  },
};
</script>

<style scoped>
.container {
  height: 100vh;
  display: flex;
  justify-content: center;
  padding: 8%;
  /* Asegúrate de que el contenedor ocupe el 100% del ancho */
}

.titulo {
  font-family: 'Lucida Sans', 'Lucida Sans Regular', 'Lucida Grande', 'Lucida Sans Unicode', Geneva, Verdana, sans-serif;
  font-style: bold;
  font-size: 35px;
  display: flex;
  flex-direction: column;
}

.seleccionar {
  font-family: 'Lucida Sans', 'Lucida Sans Regular', 'Lucida Grande', 'Lucida Sans Unicode', Geneva, Verdana, sans-serif;
  font-style: bold;
  display: flex;
  flex-direction: row;
  align-items: center;
  height: 10%;
  gap: 5%;
  padding-bottom: 4%;
}

.btn {
  border: solid 2px black;
  padding: 1%;
  /* Asegúrate de que el padding no afecte el ancho */
}

.botones {
  display: flex;
  flex-direction: row;
  /* Cambia a columna si quieres que los botones estén en una columna */
  gap: 5%;
  /* Espacio entre botones */
  width: 100%;
  /* Asegúrate de que el contenedor ocupe el 100% */
}

.modal-content {
  width: 480px;
  height: 500px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-evenly;
  background-color: #2aac4b;
  border-radius: 3%;
}

.contorno {
  background-color: white;
  height: 95%;
  width: 95%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.botones button {
  margin: 2px;
}

.btn-agregar {
  width: 100%;
  margin-bottom: 5px;
  display: flex;
  color: white;
  /* margin-left: 19px; */
  align-items: center;
  flex-direction: row;
  justify-content: space-around;
}

.body {
  padding: 30px;
  margin: 0;
  text-transform: capitalize;
}

.containerBoton {
  display: flex;
  justify-content: center;
}

hr {
  background-color: green;
  height: 2px;
  border: none;
  width: 363px;
  margin-bottom: 1%;
}

.containerError {
  background-color: rgba(255, 0, 0, 0.429);
  padding: 15px;
  text-align: center;
  font-family: "Letra";
  font-weight: bold;
  width: 310px;
  border: 3px solid red;
  margin-bottom: 5px;
  height: 180px;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 80px;
}

.containerError h4 {
  font-size: 25px;
  margin: 0;
  padding: 0;
}

h1 {
  font-family: "Letra";
  text-align: center;
  margin: 0;
  align-items: center;
  margin-top: 2%;
}

.text-h6 {
  font-size: 28px;
  font-family: "Letra";
  margin-bottom: 10px;
}

.botones .edi {
  border: none;
  border-radius: 5px;
  cursor: pointer;
  padding: 7px;
  background-color: transparent;
}

.botones .edi:hover {
  transform: scale(1.05);
  transition: all 0.5s;
}

.botones .act {
  border: none;
  border-radius: 5px;
  cursor: pointer;
  padding: 7px;
  background-color: transparent;
}

.act i {
  font-size: 22px;
  color: green;
}

.inac {
  border: none;
  border-radius: 5px;
  cursor: pointer;
  padding: 5px;
  margin: 0;
  background-color: transparent;
}

.botones .edi i {
  font-size: 20px;
}

.inac i {
  font-size: 25px;
  color: red;
}

.btn {
  font-family: "Letra";
  width: 100px;

  border-radius: 5px;
  border: none;

  cursor: pointer;
  background: -webkit-linear-gradient(bottom, #b95b5b, #d34646);
}
</style>