import streamlit as st
import pandas as pd
from io import StringIO
import logging

# Configurar logging
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s - %(levelname)s - %(message)s')

# Título de la aplicación
st.title("Combinador de Archivos COPARN y Desembarque")

# Definir columnas
columnas_coparn = [
    "No. SIC", "PIN Code", "No. de RoRo", "Tipo de RoRo", "Width", "Length",
    "Altura", "FE", "Operador", "POL", "POD", "Asignar buque", "Codigo de buque",
    "Ano de escala", "Secuencia de escala", "Weight", "Temp (C)", "Temp (F)",
    "IMDG", "UNNo", "Packing Grp", "Mercancia", "Anotaciones", "Fecha de recepcion",
    "Fecha de uso", "Fecha de caducidad", "Unidades anadidas", "Forwarder",
    "Transportista", "No. de remolque", "No. de booking"
]

columnas_desembarque = [
    "Origen", "Destino", "Fecha llegada", "Hora llegada", "Localizador",
    "Objeto", "Matricula", "Nombre"
]

def procesar_nombre(texto):
    """Convierte caracteres problemáticos a UTF-8 de forma robusta"""
    if pd.isna(texto):
        return ""
    try:
        # Intentar decodificar como UTF-8
        return str(texto)
    except UnicodeDecodeError:
        # Si falla, forzar codificación Latin-1 -> UTF-8 y reemplazar 0xed
        texto_latin1 = str(texto).encode("latin-1", errors="replace").decode("utf-8", errors="replace")
        return texto_latin1.replace(chr(0xED), "?")  # Reemplazar 0xed con "?"
    except Exception as e:
        logging.warning(f"Error procesando texto: {e}")
        return str(texto)

def combinar_archivos(coparn_file, desembarque_file):
    try:
        # Leer archivo COPARN
        df_coparn = pd.read_excel(
            coparn_file,
            header=None,
            names=columnas_coparn,
            dtype={"No. de RoRo": "string"},
            engine="openpyxl"
        )
        
        # Leer archivo Desembarque con procesamiento de texto
        df_desembarque = pd.read_excel(
            desembarque_file,
            header=None,
            names=columnas_desembarque,
            dtype={"Matricula": "string", "Nombre": "string"},
            engine="openpyxl",
            converters={"Nombre": procesar_nombre}  # Conversión directa aquí
        )

        # Depuración: Imprimir el valor problemático en la posición 209
        if len(df_desembarque) > 209:
            valor_problematico = df_desembarque.iloc[209]["Nombre"]
            logging.debug(f"Valor en la posición 209: {valor_problematico}")

        # Limpieza y normalización
        df_coparn["No. de RoRo"] = df_coparn["No. de RoRo"].str.strip().str.upper()
        df_desembarque["Matricula"] = df_desembarque["Matricula"].str.strip().str.upper()

        # Combinación de datos
        df_combinado = pd.merge(
            df_coparn,
            df_desembarque[["Matricula", "Nombre"]],
            left_on="No. de RoRo",
            right_on="Matricula",
            how="left"
        )

        # Resultado final
        df_final = df_combinado[["No. de RoRo", "PIN Code", "Nombre"]]
        df_final["Nombre"] = df_final["Nombre"].fillna("Sin dato")
        
        return df_final

    except Exception as e:
        logging.error(f"Error crítico: {e}")
        st.error(f"Error: {str(e)}")
        return None

# Interfaz de usuario
st.header("Subir Archivos")
coparn_file = st.file_uploader("Subir COPARN (.xlsx)", type="xlsx")
desembarque_file = st.file_uploader("Subir Desembarque (.xlsx)", type="xlsx")

if st.button("Generar Combinación"):
    if coparn_file and desembarque_file:
        with st.spinner("Procesando archivos..."):
            df_resultado = combinar_archivos(coparn_file, desembarque_file)
            if df_resultado is not None:
                st.success("¡Archivos combinados exitosamente!")
                st.dataframe(df_resultado)
                
                # Convertir DataFrame a CSV
                csv_buffer = StringIO()
                df_resultado.to_csv(csv_buffer, index=False, encoding="utf-8")
                csv_buffer.seek(0)
                
                # Botón de descarga
                st.download_button(
                    label="Descargar Resultado (.csv)",
                    data=csv_buffer,
                    file_name="Resultado_Combinado.csv",
                    mime="text/csv"
                )
    else:
        st.warning("Debes subir ambos archivos para continuar")
