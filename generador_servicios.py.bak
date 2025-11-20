import streamlit as st
import zipfile
import os
from io import BytesIO
import streamlit as st
import shutil
from zipfile import ZipFile
import tempfile
import subprocess
from docx import Document
from docx.shared import Pt
from docx.shared import RGBColor
from docx.enum.text import WD_PARAGRAPH_ALIGNMENT
from docx.oxml import OxmlElement
import xml.etree.ElementTree as ET
import gspread
import time  # Importar el módulo time
import logging
import inspect
import ast
import difflib
import glob
import base64
import sys
import xml.etree.ElementTree as ET
from collections import defaultdict
from lxml import etree
import re
from datetime import date
from collections import OrderedDict
from pathlib import Path
import uuid
import textwrap
from datetime import datetime
import io
from xml.dom import minidom
from xml.parsers.expat import ExpatError
# -------------------------------
# Funciones para generar archivos
# -------------------------------

WSDL_URI = "http://schemas.xmlsoap.org/wsdl/"
XSD_URI  = "http://www.w3.org/2001/XMLSchema"
SOAP11_URI = "http://schemas.xmlsoap.org/wsdl/soap/"
SOAP12_URI = "http://schemas.xmlsoap.org/wsdl/soap12/"
MIME_URI    = "http://schemas.xmlsoap.org/wsdl/mime/"

RESOURCE_MAP = {
    "ProxyService": {
        "ext": "ProxyService",
        "dataclass": "com.bea.wli.sb.services.impl.ProxyServiceEntryDocumentImpl"
    },
    "Pipeline": {
        "ext": "Pipeline",
        "dataclass": "com.bea.wli.sb.pipeline.config.impl.PipelineEntryDocumentImpl"
    },
    "WSDL": {
        "ext": "WSDL",
        "dataclass": "com.bea.wli.sb.resources.config.impl.WsdlEntryDocumentImpl"
    },
    "XMLSchema": {
        "ext": "XMLSchema",
        "dataclass": "com.bea.wli.sb.resources.config.impl.SchemaEntryDocumentImpl"
    },
    "XSLT": {
        "ext": "XSLT",
        "dataclass": "com.bea.wli.sb.resources.config.impl.XsltEntryDocumentImpl"
    },
    "XQuery": {
        "ext": "Xquery",
        "dataclass": "com.bea.wli.sb.resources.config.impl.XqueryEntryDocumentImpl"
    },
    "BusinessService": {
        "ext": "BusinessService",
        "dataclass": "com.oracle.xmlns.servicebus.business.config.impl.BusinessServiceEntryDocumentImpl"
    },
    "JCA": {
        "ext": "JCA",
        "dataclass": "com.bea.wli.sb.resources.config.impl.JcaEntryDocumentImpl"
    }
}

type_extensions = {
    ".wsdl": "WSDL",
    ".proxy": "ProxyService",
    ".pipeline": "Pipeline",
    ".xsd": "XMLSchema",
    ".bix": "BusinessService",
    ".xsl": "XSLT",
    ".xquery": "XQuery",
    ".jca": "JCA",
    ".dvm": "DVM"
}

def listar_archivos_jar(ruta_jar):
    with zipfile.ZipFile(ruta_jar, 'r') as jar:
        return jar.namelist()

def to_upper_snake_case(name: str) -> str:
    """
    Convierte un nombre camelCase o PascalCase a UPPER_SNAKE_CASE.
    Ej: "consultarInfoArchivoIngresoPrestamo" -> "CONSULTAR_INFO_ARCHIVO_INGRESO_PRESTAMO"
    """
    s1 = re.sub('(.)([A-Z][a-z]+)', r'\1_\2', name)
    s2 = re.sub('([a-z0-9])([A-Z])', r'\1_\2', s1)
    return s2.upper()
    
def generar_xmlns(nombre_operacion: str) -> str:
    """
    Convierte un nombre en camelCase a un identificador con la regla:
    - Tomar las primeras 3 letras de cada palabra.
    - Evitar letras consecutivas repetidas.
    - Anteponer 'ser'.
    """
    s = nombre_operacion.strip()
    
    # Divide en palabras: minúsculas, Mayúscula+minúsculas, o acrónimos
    tokens = re.findall(r'[a-z]+|[A-Z][a-z]+|[A-Z]+(?=[A-Z][a-z]|$)', s)

    parts = []
    for t in tokens:
        w = t.lower()
        parts.append(w[:3])  # Tomar siempre las primeras 3 letras

    combined = "".join(parts)

    # Eliminar letras consecutivas repetidas
    result = [combined[0]] if combined else []
    for c in combined[1:]:
        if c != result[-1]:
            result.append(c)

    return "ser" + "".join(result)

def generate_xsd(nombre_operacion, complexType, xmlns):
    return f'''<xs:schema targetNamespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/{nombre_operacion}/v1.0" elementFormDefault="qualified" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:{xmlns}="http://xmlns.bancocajasocial.com/co/schemas/operacion/{nombre_operacion}/v1.0" xmlns:entcab="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0">
  <xs:import namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0" schemaLocation="../../Entidades/ComunesV2.1/Cabeceras.xsd"/>
  <xs:element name="{nombre_operacion}Request" type="{xmlns}:{complexType}Request"/>
  <xs:element name="{nombre_operacion}Response" type="{xmlns}:{complexType}Response"/>
  <xs:complexType name="{complexType}Request">
    <xs:sequence>
        <xs:element name="cabeceraEntrada" type="entcab:CabeceraEntrada" minOccurs="0" maxOccurs="1"/>
    </xs:sequence>
  </xs:complexType>
  <xs:complexType name="{complexType}Response">
    <xs:sequence>
        <xs:element name="cabeceraSalida" type="entcab:CabeceraSalida" minOccurs="0" maxOccurs="1"/>
    </xs:sequence>
  </xs:complexType>
</xs:schema>'''

def capitalizar_inicio(texto: str) -> str:
    """
    Convierte la primera letra en mayúscula
    sin alterar el resto del texto.
    """
    if not texto:
        return texto
    return texto[0].upper() + texto[1:]

def obtener_wsdl_asociados(jar_file, proxy_path):
    """
    Busca referencias a WSDL dentro de un archivo .proxy del JAR.
    Retorna una lista de rutas WSDL encontradas en etiquetas <con:wsdl ref="...">.
    """
    wsdl_refs = ""

    try:
        with zipfile.ZipFile(jar_file, "r") as jar:
            contenido_proxy = jar.read(proxy_path).decode("utf-8")
            root = ET.fromstring(contenido_proxy)

            # Buscar específicamente las etiquetas <con:wsdl ref="...">
            for elem in root.iter():
                # Verifica si es una etiqueta con nombre que termina en 'wsdl'
                if elem.tag.endswith("wsdl") and "ref" in elem.attrib:
                    ref = elem.attrib["ref"]
                    wsdl_refs = ref

    except Exception as e:
        print(f"❌ Error al procesar proxy: {e}")

    return wsdl_refs
    
def leer_wsdl(jar_file, wsdl_path):
    """
    Lee un archivo WSDL desde un .jar según la ruta indicada (wsdl_path).
    Retorna el contenido completo del WSDL como texto.
    """
    wsdl_text = ""

    try:
        with zipfile.ZipFile(jar_file, "r") as jar:
            if wsdl_path in jar.namelist():
                wsdl_text = jar.read(wsdl_path).decode("utf-8")
            else:
                print(f"⚠️ No se encontró la ruta {wsdl_path} en el JAR.")

    except Exception as e:
        print(f"❌ Error al procesar el WSDL {wsdl_path}: {e}")

    return wsdl_text
    
def leer_pipeline(jar_file, pipeline_path):
    """
    Lee un archivo WSDL desde un .jar según la ruta indicada (wsdl_path).
    Retorna el contenido completo del WSDL como texto.
    """
    pipeline_text = ""

    try:
        with zipfile.ZipFile(jar_file, "r") as jar:
            if pipeline_path in jar.namelist():
                pipeline_text = jar.read(pipeline_path).decode("utf-8")
            else:
                print(f"⚠️ No se encontró la ruta {pipeline_path} en el JAR.")

    except Exception as e:
        print(f"❌ Error al procesar el WSDL {pipeline_path}: {e}")

    return pipeline_text
    
def limpiar_wsdl_contenido(wsdl_text):
    """
    Extrae únicamente el contenido del bloque CDATA (desde <definitions> hasta </definitions>)
    si el WSDL viene envuelto en un <con:wsdlEntry>.
    """
    import re

    # Buscar bloque CDATA
    match = re.search(r'<!\[CDATA\[(.*)\]\]>', wsdl_text, re.DOTALL)
    if match:
        return match.group(1).strip()
    
    # Si no tiene CDATA, devolver tal cual
    return wsdl_text.strip()

def append_in_order(parent, new_elem, after_tags):
    """
    Inserta `new_elem` dentro de `parent` inmediatamente
    después del último hijo cuyo tag esté en after_tags.
    Si no encuentra coincidencia, lo agrega al inicio.
    """
    pos = 0
    for i, child in enumerate(parent):
        # Si este hijo es uno de los que respetamos como "ancla"
        if any(child.tag.endswith(tag) for tag in after_tags):
            pos = i + 1
    parent.insert(pos, new_elem)

def fusionar_definitions(original_wsdl: str, nuevo_wsdl: str) -> str:
    # Extraer la cabecera <definitions ...>
    match_orig = re.search(r"(<definitions[^>]*>)", original_wsdl)
    match_new  = re.search(r"(<definitions[^>]*>)", nuevo_wsdl)
    if match_orig and match_new:
        return nuevo_wsdl.replace(match_new.group(1), match_orig.group(1), 1)
    return nuevo_wsdl

def reordenar_definitions(root):
    """
    Reordena los hijos de <definitions> según el orden lógico esperado.
    """
    WSDL = "{%s}" % WSDL_URI
    XSD  = "{%s}" % XSD_URI
    SOAP = "{%s}" % SOAP11_URI

    orden = [
        f"{WSDL}types",
        f"{WSDL}message",
        f"{WSDL}portType",
        f"{WSDL}binding",
        f"{WSDL}service"  # opcional, por si aparece
    ]

    # Guardar los hijos en buckets
    buckets = {tag: [] for tag in orden}
    otros = []
    for child in list(root):
        if child.tag in buckets:
            buckets[child.tag].append(child)
        else:
            otros.append(child)  # por si hay extensiones

    # Vaciar <definitions>
    for child in list(root):
        root.remove(child)

    # Reconstruir en orden
    for tag in orden:
        for elem in buckets[tag]:
            root.append(elem)

    # Dejar los "otros" al final para no perder nada
    for elem in otros:
        root.append(elem)

    return root

def aplicar_indent_local(elem, nivel=1, espacio="  "):
    """
    Aplica indentación SOLO a este elemento y sus hijos inmediatos,
    sin tocar el resto del documento.
    """
    salto = "\n" + (nivel * espacio)

    if isinstance(elem, str):
        return

    hijos = list(elem)
    if hijos:
        # Abrir bloque
        if not elem.text or not elem.text.strip():
            elem.text = salto + espacio

        for hijo in hijos:
            aplicar_indent_local(hijo, nivel + 1, espacio)

        # 👇 Asegurar salto después del ÚLTIMO hijo (antes de cerrar el padre)
        if not hijos[-1].tail or not hijos[-1].tail.strip():
            hijos[-1].tail = salto
    else:
        # Si no tiene hijos, solo controlar salto de cierre
        if not elem.text or not elem.text.strip():
            elem.text = ""
    
    # 👇 Muy importante: asegurar salto ANTES de cerrar este tag
    if not elem.tail or not elem.tail.strip():
        elem.tail = "\n" + ((nivel - 1) * espacio)

def procesar_wsdl(wsdl_content: str, wsdl_path: str,
                  target_namespace: str, xsd_path: str,
                  operation_name: str, input_msg: str, output_msg: str,
                  ns_elem_prefix: str) -> str:
    # 1) Insertar el nuevo namespace al texto plano
    
    nuevo_xmlns = f'xmlns:{ns_elem_prefix}="{target_namespace}"'
    
    #st.write(f"{nuevo_xmlns}")
    
    #aplicar_indent_local(nuevo_xmlns, nivel=2)
    
    wsdl_content_mod = agregar_namespace_texto(wsdl_content, nuevo_xmlns)

    # 2) Insertar la nueva operación (ya no tocamos <definitions>)
    wsdl_content_final = agregar_operacion_wsdl(
        wsdl_content_mod, wsdl_path, target_namespace, xsd_path,
        operation_name, input_msg, output_msg, ns_elem_prefix
    )
    wsdl_content_final = fusionar_definitions(wsdl_content_mod, wsdl_content_final)

    return wsdl_content_final

def agregar_namespace_texto(wsdl_content: str, nuevo_xmlns: str) -> str:
    """
    Inserta un nuevo xmlns dentro de <definitions ...> sin alterar el resto.
    """
    patron = r'(<definitions[^>]*)(>)'
    reemplazo = r'\1 ' + nuevo_xmlns + r'\2'
    return re.sub(patron, reemplazo, wsdl_content, count=1)

def agregar_operacion_wsdl(wsdl_content, wsdl_path, target_namespace, xsd_path,
                           operation_name, input_msg, output_msg, ns_elem_prefix):

    """
    Modifica el WSDL en memoria y devuelve el nuevo contenido como string.
    """
    # --- 1️⃣ Limpieza básica del texto XML ---
    if not wsdl_content or "<definitions" not in wsdl_content:
        raise ValueError("El WSDL recibido está vacío o no contiene <definitions>.")

    # Eliminar BOM si existe
    wsdl_content = wsdl_content.lstrip("\ufeff")

    # Reemplazar entidades problemáticas (& sueltas)
    wsdl_content = re.sub(r'&(?!amp;|lt;|gt;|quot;|apos;)', '&amp;', wsdl_content)
    
    # Extraer el bloque completo de apertura de <definitions ...>
    match = re.search(r'(<definitions\b[^>]*>)', wsdl_content)
    if match:
        defs_tag = match.group(1)
        seen_prefixes = set()
        cleaned_attrs = []
        # Buscar todos los xmlns:*="..." dentro del tag
        for attr in re.findall(r'xmlns:[a-zA-Z0-9_-]+="[^"]+"', defs_tag):
            prefix = attr.split("=")[0]
            if prefix in seen_prefixes:
                defs_tag = defs_tag.replace(attr, "", 1)  # elimina solo una ocurrencia
            else:
                seen_prefixes.add(prefix)
        # Limpieza de espacios dobles y cierre garantizado con ">"
        defs_tag = re.sub(r'\s{2,}', ' ', defs_tag).replace(' >', '>')
        if not defs_tag.endswith('>'):
            defs_tag += '>'
        # Reemplazar en el XML original
        wsdl_content = wsdl_content.replace(match.group(1), defs_tag, 1)

    # --- 2️⃣ Intentar parsear ---
    try:
        root = ET.fromstring(wsdl_content)
    except ET.ParseError as e:
        st.error(f"❌ Error al parsear el XML en agregar_operacion_wsdl: {e}")
        st.code(wsdl_content[:10000], language="xml")
        raise
    #root = tree.getroot()

    ns = {
        'wsdl': 'http://schemas.xmlsoap.org/wsdl/',
        'xs': 'http://www.w3.org/2001/XMLSchema',
        'soap': 'http://schemas.xmlsoap.org/wsdl/soap/',
    }
    
    # --- Parsear
    #root = ET.fromstring(wsdl_content)

    # Helpers de QName
    WSDL = "{%s}" % WSDL_URI
    XSD  = "{%s}" % XSD_URI
    SOAP = "{%s}" % SOAP11_URI

    # 1) Agregar import al FINAL de <types>
    types = root.find(f"{WSDL}types")
    if types is None:
        types = ET.SubElement(root, f"{WSDL}types")

    wsdl_dir = os.path.dirname(wsdl_path)
    rel_path_str = os.path.relpath(xsd_path, wsdl_dir).replace("\\", "/")

    schema = ET.Element(f"{XSD}schema")
    attribs = OrderedDict()
    attribs["schemaLocation"] = rel_path_str
    attribs["namespace"] = target_namespace
    ET.SubElement(schema, f"{XSD}import", attrib=attribs)
    
    # 👇 Aplicar indentación SOLO a este bloque
    aplicar_indent_local(schema, nivel=2)
    types.append(schema)

    # 2) Mensajes
    msg_in  = ET.SubElement(root, f"{WSDL}message", {"name": input_msg})
    ET.SubElement(msg_in, f"{WSDL}part", {"name": input_msg, "element": f"{ns_elem_prefix}:{input_msg}"})
    aplicar_indent_local(msg_in, nivel=2)
    
    msg_out = ET.SubElement(root, f"{WSDL}message", {"name": output_msg})
    ET.SubElement(msg_out, f"{WSDL}part", {"name": output_msg, "element": f"{ns_elem_prefix}:{output_msg}"})
    aplicar_indent_local(msg_out, nivel=2)
    
    # 3) PortType
    port_type = root.find(f"{WSDL}portType")
    if port_type is None:
        port_type = ET.SubElement(root, f"{WSDL}portType", {"name": f"{operation_name}_Port"})
    op = ET.SubElement(port_type, f"{WSDL}operation", {"name": operation_name})
    
    ET.SubElement(op, f"{WSDL}input",  {"message": f"tns:{input_msg}"})
    ET.SubElement(op, f"{WSDL}output", {"message": f"tns:{output_msg}"})
    aplicar_indent_local(op, nivel=2)

    # 4) Binding
    binding = root.find(f"{WSDL}binding")
    if binding is None:
        binding = ET.SubElement(root, f"{WSDL}binding", {"name": f"{operation_name}_Binding",
                                                         "type": f"tns:{port_type.get('name')}"} )
        ET.SubElement(binding, f"{SOAP}binding", {"style": "document",
                                                  "transport": "http://schemas.xmlsoap.org/soap/http"})

    opb = ET.SubElement(binding, f"{WSDL}operation", {"name": operation_name})
    ET.SubElement(opb, f"{SOAP}operation", {"style": "document", "soapAction": target_namespace})
    inp = ET.SubElement(opb, f"{WSDL}input")
    out = ET.SubElement(opb, f"{WSDL}output")

    # Detectar el prefijo correcto para <body>
    # Buscamos un <body> existente en otro binding para usar el mismo prefijo
    existing_body = root.find(".//{http://schemas.xmlsoap.org/wsdl/soap/}body")
    if existing_body is not None:
        body_tag = existing_body.tag  # Esto conserva el namespace y prefijo original
    else:
        body_tag = f"{SOAP}body"

    ET.SubElement(inp, body_tag, {"use": "literal", "parts": input_msg})
    ET.SubElement(out, body_tag, {"use": "literal", "parts": output_msg})
    
    aplicar_indent_local(opb, nivel=2)
    # Reordenar antes de devolver
    root = reordenar_definitions(root)
    wsdl_str = ET.tostring(root, encoding="unicode")
    
    # --- REEMPLAZO de nsX: por soap12: ---
    wsdl_str = re.sub(r'\bns\d+:', 'soap12:', wsdl_str)

    return wsdl_str
    
def crear_wsdl_exp(service_name: str,
                    wsdl_path: str,
                    xsd_path: str,
                    operation_name: str,
                    input_msg: str,
                    output_msg: str,
                    ns_elem_prefix: str,
                    target_namespace_xsd: str,
                    write_to_file: bool = False) -> str:

    import xml.etree.ElementTree as ET
    import os
    from collections import OrderedDict

    def _relpath_from_wsdl(wsdl_path: str, xsd_path: str) -> str:
        wsdl_path = os.path.normpath(wsdl_path)
        xsd_path  = os.path.normpath(xsd_path)
        wsdl_dir = os.path.dirname(wsdl_path)
        try:
            rel = os.path.relpath(xsd_path, wsdl_dir)
        except ValueError:
            rel = xsd_path
        return rel.replace("\\", "/")

    # targetNamespace del WSDL
    tns_wsdl = f"http://xmlns.bancocajasocial.com/co/servicios/{service_name}/v1.0"

    # Registrar solo los que queremos que ElementTree use en nodos
    ET.register_namespace("", WSDL_URI)
    ET.register_namespace("soap", SOAP11_URI)
    ET.register_namespace("xsd", XSD_URI)

    # Atributos adicionales que sí queremos forzar en <definitions>
    defs_attribs = OrderedDict()
    defs_attribs["xmlns"] = WSDL_URI   # 👈 agregado xmlns explícito
    defs_attribs["targetNamespace"] = tns_wsdl
    defs_attribs["xmlns:tns"] = tns_wsdl
    defs_attribs["xmlns:soap12"] = SOAP12_URI
    defs_attribs["xmlns:mime"] = MIME_URI
    defs_attribs[f"xmlns:{ns_elem_prefix}"] = target_namespace_xsd

    definitions = ET.Element("definitions", attrib=defs_attribs)

    # <types>
    types = ET.SubElement(definitions, "types")
    ET.SubElement(types, f"{{{XSD_URI}}}schema", {
        "targetNamespace": f"{tns_wsdl}/types",
        "elementFormDefault": "qualified"
    })
    schema_import_block = ET.SubElement(types, f"{{{XSD_URI}}}schema")
    rel_xsd = _relpath_from_wsdl(wsdl_path, xsd_path)
    ET.SubElement(schema_import_block, f"{{{XSD_URI}}}import", {
        "schemaLocation": rel_xsd,
        "namespace": target_namespace_xsd
    })

    # messages
    msg_in = ET.SubElement(definitions, "message", {"name": input_msg})
    ET.SubElement(msg_in, "part", {"name": input_msg, "element": f"{ns_elem_prefix}:{input_msg}"})
    msg_out = ET.SubElement(definitions, "message", {"name": output_msg})
    ET.SubElement(msg_out, "part", {"name": output_msg, "element": f"{ns_elem_prefix}:{output_msg}"})

    # portType
    port_type_name = f"{operation_name}_Port"
    portType = ET.SubElement(definitions, "portType", {"name": port_type_name})
    op = ET.SubElement(portType, "operation", {"name": operation_name})
    ET.SubElement(op, "input",  {"message": f"tns:{input_msg}"})
    ET.SubElement(op, "output", {"message": f"tns:{output_msg}"})

    # binding
    binding_name = f"{operation_name}_Binding"
    binding = ET.SubElement(definitions, "binding", {"name": binding_name, "type": f"tns:{port_type_name}"})
    ET.SubElement(binding, f"{{{SOAP11_URI}}}binding", {
        "style": "document",
        "transport": "http://schemas.xmlsoap.org/soap/http"
    })
    bop = ET.SubElement(binding, "operation", {"name": operation_name})
    ET.SubElement(bop, f"{{{SOAP11_URI}}}operation", {
        "style": "document",
        "soapAction": tns_wsdl+"/"+operation_name
    })
    binp = ET.SubElement(bop, "input")
    ET.SubElement(binp, f"{{{SOAP11_URI}}}body", {"use": "literal", "parts": input_msg})
    bout = ET.SubElement(bop, "output")
    ET.SubElement(bout, f"{{{SOAP11_URI}}}body", {"use": "literal", "parts": output_msg})

    wsdl_str = ET.tostring(definitions, encoding="unicode")

    if write_to_file:
        dirpath = os.path.dirname(wsdl_path)
        if dirpath and not os.path.exists(dirpath):
            os.makedirs(dirpath, exist_ok=True)
        ET.ElementTree(definitions).write(wsdl_path, encoding="utf-8", xml_declaration=True)

    return wsdl_str

def obtener_namespace_y_binding(wsdl_content: str) -> tuple[str, str]:
    """
    Extrae el targetNamespace y el nombre del binding de un WSDL.

    Args:
        wsdl_content (str): Contenido del archivo WSDL como string.

    Returns:
        tuple[str, str]: (targetNamespace, nombre_binding)
    """
    # Parsear el contenido del WSDL
    root = ET.fromstring(wsdl_content)

    # Namespace WSDL
    wsdl_ns = "{http://schemas.xmlsoap.org/wsdl/}"

    # Obtener targetNamespace
    target_namespace = root.attrib.get("targetNamespace", "")

    # Buscar primer binding
    binding = root.find(f"{wsdl_ns}binding")
    binding_name = binding.attrib.get("name", "") if binding is not None else ""

    return target_namespace, binding_name

def quitar_extension(ruta: str) -> str:
    return str(Path(ruta).with_suffix(""))

def crear_proxy_exp(proxy_name: str,
                    wsdl_ref: str,
                    binding_wsdl: str,
                    namespace_wsdl: str,
                    pipeline_ref: str,
                    endpoint: str,
                    write_to_file: bool = False) -> str:
    """
    Crea un archivo proxy .proxy personalizado en formato XML para OSB.

    - proxy_name: Nombre del servicio/proxy
    - wsdl_ref: Ruta del WSDL (sin extensión .wsdl)
    - namespace: Namespace usado para el binding
    - pipeline_ref: Ruta del pipeline (sin extensión .pipeline)
    - endpoint: Endpoint expuesto (/NombreDelServicio)
    """
    
    # 🔑 Normalizar paths a formato OSB (/ en vez de \)
    wsdl_ref = wsdl_ref.replace("\\", "/")
    pipeline_ref = pipeline_ref.replace("\\", "/")


    xml_content = f'''<?xml version="1.0" encoding="UTF-8"?>
<ser:proxyServiceEntry
    xmlns:ser="http://www.bea.com/wli/sb/services"
    xmlns:con="http://www.bea.com/wli/sb/services/security/config"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:oper="http://xmlns.oracle.com/servicebus/proxy/operations"
    xmlns:tran="http://www.bea.com/wli/sb/transports"
    xmlns:env="http://www.bea.com/wli/config/env">
    <ser:coreEntry isAutoPublish="false">>
        <ser:security>
            <con:inboundWss processWssHeader="true"/>
        </ser:security>
        <ser:binding type="SOAP" xsi:type="con:SoapBindingType" isSoap12="false"
            xmlns:con="http://www.bea.com/wli/sb/services/bindings/config">
            <con:wsdl ref="{wsdl_ref}"/>
            <con:binding>
                <con:name>{binding_wsdl}</con:name>
                <con:namespace>{namespace_wsdl}</con:namespace>
            </con:binding>
            <con:selector type="SOAP body"/>
        </ser:binding>
        <oper:operations enabled="true"/>
        <ser:ws-policy>
            <ser:binding-mode>no-policies</ser:binding-mode>
        </ser:ws-policy>
        <ser:invoke ref="{pipeline_ref}" xsi:type="con:PipelineRef"
            xmlns:con="http://www.bea.com/wli/sb/pipeline/config"/>
        <ser:xqConfiguration>
            <ser:snippetVersion>1.0</ser:snippetVersion>
        </ser:xqConfiguration>
    </ser:coreEntry>
    <ser:endpointConfig>
        <tran:provider-id>http</tran:provider-id>
        <tran:inbound>true</tran:inbound>
        <tran:URI>
            <env:value>/{endpoint}</env:value>
        </tran:URI>
        <tran:inbound-properties/>
        <tran:provider-specific xsi:type="http:HttpEndPointConfiguration"
            xmlns:http="http://www.bea.com/wli/sb/transports/http">
            <http:inbound-properties/>
            <http:compression>
                <http:compression-support>false</http:compression-support>
            </http:compression>
        </tran:provider-specific>
    </ser:endpointConfig>
</ser:proxyServiceEntry>'''

    if write_to_file:
        file_name = f"{proxy_name}.proxy"
        with open(file_name, "w", encoding="utf-8") as f:
            f.write(xml_content)

    return xml_content

def crear_pipeline_exp(pipeline_name: str,
                       wsdl_ref: str,
                       binding_wsdl: str,
                       namespace_wsdl: str,
                       operation_name: str,
                       service_target_ref: str,
                       write_to_file: bool = False) -> str:
    """
    Crea un archivo .pipeline en formato XML para OSB (versión simplificada y parametrizable).

    Parámetros:
    - pipeline_name: Nombre del pipeline (archivo .pipeline)
    - wsdl_ref: Ruta del WSDL (sin extensión .wsdl)
    - binding_wsdl: Nombre del binding en el WSDL
    - namespace_wsdl: Namespace usado en el binding
    - operation_name: Nombre de la operación expuesta
    - service_target_ref: Referencia al servicio/proxy destino
    - write_to_file: Si True, escribe el archivo .pipeline
    """

    # Normalizar rutas OSB (/ en lugar de \)
    wsdl_ref = wsdl_ref.replace("\\", "/")
    service_target_ref = quitar_extension(service_target_ref)
    service_target_ref = service_target_ref.replace("\\", "/")

    xml_content = f'''<?xml version="1.0" encoding="UTF-8"?>
    <con:pipelineEntry xmlns:con="http://www.bea.com/wli/sb/pipeline/config"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:con1="http://www.bea.com/wli/sb/stages/config"
        xmlns:con2="http://www.bea.com/wli/sb/stages/routing/config">
        <con:coreEntry>
            <con:binding type="SOAP" isSoap12="false" xsi:type="con:SoapBindingType">
                <con:wsdl ref="{wsdl_ref}"/>
                <con:binding>
                    <con:name>{binding_wsdl}</con:name>
                    <con:namespace>{namespace_wsdl}</con:namespace>
                </con:binding>
            </con:binding>
            <con:xqConfiguration>
                <con:snippetVersion>1.0</con:snippetVersion>
            </con:xqConfiguration>
        </con:coreEntry>
        <con:router>
            <con:pipeline type="request" name="{operation_name}_request-ad48654.N5fb25bd0.0.198ed0a6781.N7fcf">
                <con:stage id="_StageId-ad48654.N5fb25bd0.0.198ed0a6781.N7fcd" name="stg_inicializarVariables">
                    <con:context/>
                    <con:actions>
                        <con1:assign varName="operacionExp">
                            <con1:expr>
                                <con1:xqueryText>&lt;operacion>{operation_name}&lt;/operacion></con1:xqueryText>
                            </con1:expr>
                        </con1:assign>
                    </con:actions>
                </con:stage>
            </con:pipeline>
            <con:pipeline type="response" name="{operation_name}_response-ad48654.N5fb25bd0.0.198ed0a6781.N7fce">
                <con:stage id="_StageId-ad48654.N5fb25bd0.0.198ed0a6781.N7fcc" name="stg_responder">
                    <con:context/>
                    <con:actions>
                        <con1:reply/>
                    </con:actions>
                </con:stage>
            </con:pipeline>
            <con:flow>
                <con:branch-node type="operation" id="_FlowId-ad48654.N5fb25bd0.0.198ed0a6781.N7fcb" name="BN_EXP">
                    <con:context/>
                    <con:branch-table>
                        <con:branch name="{operation_name}">
                            <con:operator>equals</con:operator>
                            <con:value/>
                            <con:flow>
                                <con:pipeline-node name="{operation_name}_PNN_EXP">
                                    <con:request>{operation_name}_request-ad48654.N5fb25bd0.0.198ed0a6781.N7fcf</con:request>
                                    <con:response>{operation_name}_response-ad48654.N5fb25bd0.0.198ed0a6781.N7fce</con:response>
                                </con:pipeline-node>
                                <con:route-node name="{operation_name}_RN">
                                    <con:context/>
                                    <con:actions>
                                        <con2:route>
                                            <con2:service ref="{service_target_ref}" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                                            <con2:operation>{operation_name}</con2:operation>
                                            <con2:outboundTransform/>
                                            <con2:responseTransform/>
                                        </con2:route>
                                    </con:actions>
                                </con:route-node>
                            </con:flow>
                        </con:branch>
                        <con:default-branch>
                            <con:flow/>
                        </con:default-branch>
                    </con:branch-table>
                </con:branch-node>
            </con:flow>
        </con:router>
    </con:pipelineEntry>'''

    if write_to_file:
        file_name = f"{pipeline_name}.pipeline"
        with open(file_name, "w", encoding="utf-8") as f:
            f.write(xml_content)

    return xml_content

def prettify(xml_string: str) -> str:
    try:
        parsed = minidom.parseString(xml_string)
        return parsed.toprettyxml(indent="  ", encoding="utf-8").decode("utf-8")
    except ExpatError as e:
        # Log en consola
        print("❌ Error en prettify:", e)
        print("XML recibido:\n", xml_string)

        # Mostrar en la app para depurar
        st.error(f"❌ Error al formatear XML: {e}")
        with st.expander("📄 XML crudo con error", expanded=False):
            st.code(xml_string, language="xml")

        # Devuelvo el XML tal cual, sin formatear, para no romper el flujo
        return xml_string

def pretty_print_xml(xml_str):
    # Parsear el XML
    root = ET.fromstring(xml_str)

    # Función recursiva para aplicar indentación
    def indent(elem, level=0):
        i = "\n" + level * "\t"
        if len(elem):
            if not elem.text or not elem.text.strip():
                elem.text = i + "\t"
            for child in elem:
                indent(child, level + 1)
            if not child.tail or not child.tail.strip():
                child.tail = i
        if level and (not elem.tail or not elem.tail.strip()):
            elem.tail = i

    indent(root)

    return '<?xml version="1.0" encoding="UTF-8"?>\n' + ET.tostring(root, encoding="unicode")

def generar_ids(op_name):
    """Genera IDs únicos basados en la operación"""
    uid = uuid.uuid4().hex[:12]
    request_id = f"request-{uid}"
    response_id = f"response-{uid}"
    action_id = f"_ActionId-{uid}"
    return request_id, response_id, action_id

def insertar_branch(pipeline_text, nuevo_branch):
    # Buscar indentación del <con:default-branch>
    match = re.search(r'(\s*)<con:default-branch>', pipeline_text)
    indent = match.group(1) if match else "    "  # fallback 4 espacios

    # Limpiar indentación y espacios extra
    nuevo_branch = textwrap.dedent(nuevo_branch).strip("\n")

    # Re-indentamos el bloque
    branch_lines = [indent + line for line in nuevo_branch.split("\n")]
    nuevo_branch_indented = "".join(branch_lines)

    # Insertamos SIN agregar salto adicional (usamos solo el del archivo original)
    pipeline_text = re.sub(
        r'(\s*<con:default-branch>)',
        nuevo_branch_indented + r"\1",   # <<-- sin "\n"
        pipeline_text,
        1
    )

    return pipeline_text

def agregar_operacion_pipeline(pipeline_text, op_name, targetnamespace,ubicacion_xsd_exp,ubicacion_proxy_abc):
    request_id, response_id, action_id = generar_ids(op_name)

    # 1. Crear pipelines de request/response
    nuevos_pipelines = f"""<con:pipeline name="{op_name}-request-ad48659.N6b80025f.0.198e784c905.N7f59" type="request">
	<con:stage name="stg_inicializarVariables" id="_StageId-ad48659.N6b80025f.0.198e784c905.N7f58">
		<con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config"/>
		<con:actions>
			<con3:assign varName="operacionExp" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
				<con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f57</con2:id>
				<con3:expr>
					<con2:xqueryText>&lt;operacion>{{$operation}}&lt;/operacion></con2:xqueryText>
				</con3:expr>
			</con3:assign>
			<con5:assign varName="fechaHoraInicio" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
				<con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f56</con2:id>
				<con5:expr>
					<con2:xqueryText>fn:substring( fn:replace(fn:string( fn:current-dateTime()) , "T" ," ") ,0, 22)</con2:xqueryText>
				</con5:expr>
			</con5:assign>
			<con3:assign varName="espacioNombreEXP" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
				<con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f55</con2:id>
				<con3:expr>
					<con2:xqueryText>fn:namespace-uri($body/*)</con2:xqueryText>
				</con3:expr>
			</con3:assign>
			<con3:assign varName="prefijoEXP" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
				<con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f54</con2:id>
				<con3:expr>
					<con2:xqueryText>"srvexp"</con2:xqueryText>
				</con3:expr>
			</con3:assign>
		</con:actions>
	</con:stage>
	<con:stage name="stg_respaldarEntrada" id="_StageId-ad48659.N6b80025f.0.198e784c905.N7f53">
		<con:context>
			<con1:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarObligacionesConvenioMasivo/v1.0"/>
			<con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarValidacionesPantallaVisacion/v1.0"/>
			<con1:userNsDecl prefix="v13" namespace="{targetnamespace}"/>
			<con1:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInformacionVisacion/v1.0"/>
		</con:context>
		<con:actions>
			<con3:assign varName="mensajeEntradaExp" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
				<con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f52</con2:id>
				<con3:expr>
					<con2:xqueryText>$body/v13:{op_name}Request</con2:xqueryText>
				</con3:expr>
			</con3:assign>
		</con:actions>
	</con:stage>
	<con:stage name="stg_validarEntrada" id="_StageId-ad48659.N6b80025f.0.198e784c905.N7f51">
		<con:context>
			<con1:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarObligacionesConvenioMasivo/v1.0"/>
			<con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarValidacionesPantallaVisacion/v1.0"/>
			<con1:userNsDecl prefix="v13" namespace="{targetnamespace}"/>
			<con1:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInformacionVisacion/v1.0"/>
		</con:context>
		<con:actions>
			<con3:validate xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
				<con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f50</con2:id>
				<con3:schema ref="{ubicacion_xsd_exp}"/>
				<con3:schemaElement xmlns:v1="{targetnamespace}">v1:{op_name}Request</con3:schemaElement>
				<con3:varName>body</con3:varName>
				<con3:location>
					<con2:xpathText>./v13:{op_name}Request</con2:xpathText>
				</con3:location>
			</con3:validate>
		</con:actions>
	</con:stage>
    </con:pipeline>
    <con:pipeline name="{op_name}-response-ad48659.N6b80025f.0.198e784c905.N7f4f" type="response">
        <con:stage name="stg_validarSalida" id="_StageId-ad48659.N6b80025f.0.198e784c905.N7f4e">
            <con:context>
                <con1:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarObligacionesConvenioMasivo/v1.0"/>
                <con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarValidacionesPantallaVisacion/v1.0"/>
                <con1:userNsDecl prefix="v13" namespace="{targetnamespace}"/>
                <con1:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInformacionVisacion/v1.0"/>
            </con:context>
            <con:actions>
                <con3:validate xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                    <con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f4d</con2:id>
                    <con3:schema ref="{ubicacion_xsd_exp}"/>
                    <con3:schemaElement xmlns:v1="{targetnamespace}">v1:{op_name}Response</con3:schemaElement>
                    <con3:varName>body</con3:varName>
                    <con3:location>
                        <con2:xpathText>./v13:{op_name}Response</con2:xpathText>
                    </con3:location>
                </con3:validate>
            </con:actions>
        </con:stage>
        <con:stage name="stg_transformacionBody" id="_StageId-ad48659.N6b80025f.0.198e784c905.N7f4c">
            <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0"/>
            </con:context>
            <con:actions>
                <con5:replace varName="transformacionBody" contents-only="false" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                    <con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f4b</con2:id>
                    <con1:expr>
                        <con2:xqueryTransform>
                            <con2:resource ref="ComponentesComunes/Resources/XQUERYs/xq_Auditoria_to_RegistrarAuditoriaSOA"/>
                            <con2:param name="codigoError">
                                <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError)</con2:path>
                            </con2:param>
                            <con2:param name="horaInicialTX">
                                <con2:path>$fechaHoraInicio</con2:path>
                            </con2:param>
                            <con2:param name="mensajeResponse">
                                <con2:path>$body/*</con2:path>
                            </con2:param>
                            <con2:param name="mensajeRequest">
                                <con2:path>$mensajeEntradaExp</con2:path>
                            </con2:param>
                            <con2:param name="pId">
                                <con2:path>$messageID</con2:path>
                            </con2:param>
                            <con2:param name="nombreFlujo">
                                <con2:path>$operation</con2:path>
                            </con2:param>
                            <con2:param name="archivoResponse">
                                <con2:path>""</con2:path>
                            </con2:param>
                            <con2:param name="archivoRequest">
                                <con2:path>""</con2:path>
                            </con2:param>
                            <con2:param name="oficina">
                                <con2:path>data($mensajeEntradaExp/*:cabeceraEntrada/*:invocador/*:codigoOficina)</con2:path>
                            </con2:param>
                            <con2:param name="numeroReferencia">
                                <con2:path>data($mensajeEntradaExp/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)</con2:path>
                            </con2:param>
                            <con2:param name="tipoError">
                                <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:tipoError)</con2:path>
                            </con2:param>
                            <con2:param name="usuario">
                                <con2:path>data($mensajeEntradaExp/*:cabeceraEntrada/*:invocador/*:usuario)</con2:path>
                            </con2:param>
                            <con2:param name="id">
                                <con2:path>data($mensajeEntradaExp/*:cabeceraEntrada/*:invocador/*:identificadorTx)</con2:path>
                            </con2:param>
                            <con2:param name="descripcionError">
                                <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:descripcionError)</con2:path>
                            </con2:param>
                            <con2:param name="infoAdicional3">
                                <con2:path>" "</con2:path>
                            </con2:param>
                            <con2:param name="descripcionTipoError">
                                <con2:path>" "</con2:path>
                            </con2:param>
                            <con2:param name="infoAdicional2">
                                <con2:path>concat("NUMEROSOLICITUD :",data($mensajeEntradaExp/*:cabeceraEntrada/*:invocador/*:numeroSolicitud))</con2:path>
                            </con2:param>
                            <con2:param name="infoAdicional1">
                                <con2:path>concat("CODIGOCLIENTE :",data($mensajeEntradaExp/*:cabeceraEntrada/*:invocador/*:codigoCliente))</con2:path>
                            </con2:param>
                            <con2:param name="codigoRespuesta">
                                <con2:path>data($body/*/*:cabeceraSalida/*:tipoRespuesta)</con2:path>
                            </con2:param>
                        </con2:xqueryTransform>
                    </con1:expr>
                </con5:replace>
            </con:actions>
        </con:stage>
        <con:stage name="stg_auditarMensaje" id="_StageId-ad48659.N6b80025f.0.198e784c905.N7f4a">
            <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config"/>
            <con:actions>
                <con4:route xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                    <con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f49</con2:id>
                    <con4:service ref="UtilitariosEBS/Proxies/AuditoriaSOA/RegistrarAuditoriaSOADATV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                    <con4:operation>registrarAuditoria</con4:operation>
                    <con4:outboundTransform>
                        <con5:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7f48</con2:id>
                            <con1:expr>
                                <con2:xqueryText>$transformacionBody</con2:xqueryText>
                            </con1:expr>
                        </con5:replace>
                    </con4:outboundTransform>
                </con4:route>
            </con:actions>
        </con:stage>
    </con:pipeline>"""

    # Insertar antes de <con:flow>
    #pipeline_text = re.sub(r'(?=<con:flow)', nuevos_pipelines + "\n", pipeline_text, 1)
    pipeline_text = re.sub(r'(?=<con:flow)', lambda m: nuevos_pipelines + "\n", pipeline_text, 1)

    # 2. Crear nuevo branch con pipelines y route
    nuevo_branch = f"""
    <con:branch name="{op_name}">
        <con:operator>equals</con:operator>
        <con:value/>
        <con:flow>
            <con:pipeline-node name="{op_name}_PNN_EXP">
                <con:request>{op_name}-request-ad48659.N6b80025f.0.198e784c905.N7f59</con:request>
                <con:response>{op_name}-response-ad48659.N6b80025f.0.198e784c905.N7f4f</con:response>
            </con:pipeline-node>
            <con:route-node name="{op_name}_RN_TO_EBS">
                <con:context/>
                <con:actions>
                    <con1:route xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad48659.N6b80025f.0.198e784c905.N7ff8</con2:id>
                        <con1:service ref="{ubicacion_proxy_abc}" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                        <con1:operation>{op_name}</con1:operation>
                        <con1:outboundTransform/>
                        <con1:responseTransform/>
                    </con1:route>
                </con:actions>
            </con:route-node>
        </con:flow>
    </con:branch>"""

    # Insertar dentro de <con:branch-table>
    #pipeline_text = re.sub(r'(?=</con:branch-table>)', nuevo_branch + "\n", pipeline_text, 1)
    #pipeline_text = re.sub(r'(?=</con:branch-table>)', lambda m: nuevo_branch + "\n", pipeline_text, 1)
    # pipeline_text = re.sub(
    # r'(?=\s*<con:default-branch>)',
    # nuevo_branch.replace("\\", "\\\\"),
    # pipeline_text,
    # 1
    # )
    
    pipeline_text = insertar_branch(pipeline_text,nuevo_branch)
    
    return pipeline_text

def crear_wsdl_abc(operation_name: str,
                   wsdl_path: str,
                   xsd_path: str,
                   target_namespace_xsd: str,
                   ns_elem_prefix: str,
                   write_to_file: bool = False) -> str:
    """
    Genera un WSDL para la capa ABC (siempre con versión v1.0).

    Args:
        operation_name: Nombre de la operación (ej: "consultarInfoArchivoIngresoPrestamo").
        wsdl_path: Ruta absoluta al archivo WSDL de salida.
        xsd_path: Ruta relativa o absoluta del archivo XSD.
        target_namespace_xsd: Namespace del XSD.
        ns_elem_prefix: Prefijo para el namespace del XSD (ej: "serconsinfarchingrpres").
        write_to_file: Si True, escribe el archivo en disco.

    Returns:
        str: Contenido del WSDL generado en formato string.
    """
    import xml.etree.ElementTree as ET
    import os
    from collections import OrderedDict
    from pathlib import PurePosixPath

    # URIs
    WSDL_URI = "http://schemas.xmlsoap.org/wsdl/"
    SOAP11_URI = "http://schemas.xmlsoap.org/wsdl/soap/"
    SOAP12_URI = "http://schemas.xmlsoap.org/wsdl/soap12/"
    MIME_URI = "http://schemas.xmlsoap.org/wsdl/mime/"
    XSD_URI = "http://www.w3.org/2001/XMLSchema"

    version = "v1.0"  # fijo
    tns_wsdl = f"http://xmlns.bancocajasocial.com/co/servicios/abc/{operation_name}/{version}"

    # Calcular schemaLocation relativo desde wsdl_path hasta xsd_path (siempre con /)
    schema_location = os.path.relpath(
        xsd_path,
        start=os.path.dirname(wsdl_path)
    ).replace("\\", "/")

    # Namespaces (sin duplicar soap)
    ET.register_namespace("", WSDL_URI)
    ET.register_namespace("soap", SOAP11_URI)
    ET.register_namespace("xsd", XSD_URI)

    # Definiciones
    defs_attribs = OrderedDict()
    defs_attribs["targetNamespace"] = tns_wsdl
    defs_attribs["xmlns"] = WSDL_URI
    defs_attribs["xmlns:tns"] = tns_wsdl
    defs_attribs["xmlns:soap12"] = SOAP12_URI
    defs_attribs["xmlns:mime"] = MIME_URI
    defs_attribs[f"xmlns:{ns_elem_prefix}"] = target_namespace_xsd

    definitions = ET.Element("definitions", attrib=defs_attribs)

    # <types> con schema targetNamespace y schema import
    types = ET.SubElement(definitions, "types")
    ET.SubElement(types, f"{{{XSD_URI}}}schema", {
        "targetNamespace": f"{tns_wsdl}/types",
        "elementFormDefault": "qualified"
    })
    schema_import_block = ET.SubElement(types, f"{{{XSD_URI}}}schema")
    ET.SubElement(schema_import_block, f"{{{XSD_URI}}}import", {
        "schemaLocation": schema_location,
        "namespace": target_namespace_xsd
    })

    # mensajes
    msg_in = ET.SubElement(definitions, "message", {"name": f"{operation_name}Request"})
    ET.SubElement(msg_in, "part", {
        "name": f"{operation_name}Request",
        "element": f"{ns_elem_prefix}:{operation_name}Request"
    })

    msg_out = ET.SubElement(definitions, "message", {"name": f"{operation_name}Response"})
    ET.SubElement(msg_out, "part", {
        "name": f"{operation_name}Response",
        "element": f"{ns_elem_prefix}:{operation_name}Response"
    })

    # portType
    port_type_name = f"{to_upper_snake_case(operation_name)}_PORT"
    portType = ET.SubElement(definitions, "portType", {"name": port_type_name})
    op = ET.SubElement(portType, "operation", {"name": operation_name})
    ET.SubElement(op, "input", {"message": f"tns:{operation_name}Request"})
    ET.SubElement(op, "output", {"message": f"tns:{operation_name}Response"})

    # binding
    binding_name = f"{to_upper_snake_case(operation_name)}_Binding"
    binding = ET.SubElement(definitions, "binding", {"name": binding_name, "type": f"tns:{port_type_name}"})
    ET.SubElement(binding, f"{{{SOAP11_URI}}}binding", {
        "style": "document",
        "transport": "http://schemas.xmlsoap.org/soap/http"
    })
    bop = ET.SubElement(binding, "operation", {"name": operation_name})
    # Ajuste del soapAction: SOLO hasta /v1.0
    ET.SubElement(bop, f"{{{SOAP11_URI}}}operation", {
        "style": "document",
        "soapAction": f"{tns_wsdl}"
    })
    binp = ET.SubElement(bop, "input")
    ET.SubElement(binp, f"{{{SOAP11_URI}}}body", {
        "use": "literal", "parts": f"{operation_name}Request"
    })
    bout = ET.SubElement(bop, "output")
    ET.SubElement(bout, f"{{{SOAP11_URI}}}body", {
        "use": "literal", "parts": f"{operation_name}Response"
    })

    raw_str = ET.tostring(definitions, encoding="unicode")
    wsdl_str = prettify(raw_str)

    # Guardar en archivo
    if write_to_file:
        dirpath = os.path.dirname(wsdl_path)
        if dirpath and not os.path.exists(dirpath):
            os.makedirs(dirpath, exist_ok=True)
        ET.ElementTree(definitions).write(wsdl_path, encoding="utf-8", xml_declaration=True)

    return wsdl_str

def crear_proxy_abc(wsdl_ref: str, binding_wsdl: str, namespace_wsdl: str, pipeline_ref: str) -> str:
    """
    Crea el XML de un proxy service parametrizado con el wsdl, binding, namespace y pipeline.

    Parámetros:
        wsdl_ref (str): Ruta del WSDL en OSB (ej: "STARLT_ABC/Resources/WSDLs/CONSULTAR_INFO_GENERAL_IBR")
        binding_wsdl (str): Nombre del binding (ej: "CONSULTAR_INFO_GENERAL_IBR_Binding")
        namespace_wsdl (str): Namespace del WSDL (ej: "http://xmlns.bancocajasocial.com/co/servicios/abc/consultarInfoGeneralIbr/v1.0")
        pipeline_ref (str): Ruta del pipeline en OSB (ej: "STARLT_ABC/Pipeline/PL_CONSULTAR_INFO_GENERAL_IBRDAV1.0")

    Retorna:
        str: XML generado para el proxy
    """
    
    # 🔑 Normalizar paths a formato OSB (/ en vez de \)
    wsdl_ref = wsdl_ref.replace("\\", "/")
    pipeline_ref = pipeline_ref.replace("\\", "/")

    proxy_xml = f"""<?xml version="1.0" encoding="UTF-8"?>
    <ser:proxyServiceEntry xmlns:ser="http://www.bea.com/wli/sb/services" xmlns:con="http://www.bea.com/wli/sb/services/security/config" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:oper="http://xmlns.oracle.com/servicebus/proxy/operations" xmlns:tran="http://www.bea.com/wli/sb/transports">
        <ser:coreEntry>
            <ser:security>
                <con:inboundWss processWssHeader="true"/>
            </ser:security>
            <ser:binding type="SOAP" xsi:type="con:SoapBindingType" isSoap12="false" xmlns:con="http://www.bea.com/wli/sb/services/bindings/config">
                <con:wsdl ref="{wsdl_ref}"/>
                <con:binding>
                    <con:name>{binding_wsdl}</con:name>
                    <con:namespace>{namespace_wsdl}</con:namespace>
                </con:binding>
                <con:selector type="SOAP body"/>
            </ser:binding>
            <oper:operations enabled="true"/>
            <ser:ws-policy>
                <ser:binding-mode>no-policies</ser:binding-mode>
            </ser:ws-policy>
            <ser:invoke ref="{pipeline_ref}" xsi:type="con:PipelineRef" xmlns:con="http://www.bea.com/wli/sb/pipeline/config"/>
            <ser:xqConfiguration>
                <ser:snippetVersion>1.0</ser:snippetVersion>
            </ser:xqConfiguration>
        </ser:coreEntry>
        <ser:endpointConfig>
            <tran:provider-id>local</tran:provider-id>
            <tran:inbound>true</tran:inbound>
            <tran:inbound-properties/>
        </ser:endpointConfig>
    </ser:proxyServiceEntry>"""
    return proxy_xml

def crear_pipeline_abc(wsdl_ref: str, binding_wsdl: str, namespace_wsdl: str, operation_name: str) -> str:
    """
    Crea el XML de un pipeline parametrizado con el wsdl, binding y namespace.

    Parámetros:
        wsdl_ref (str): Ruta del WSDL en OSB (ej: "STARLT_ABC/Resources/WSDLs/CONSULTAR_INFO_GENERAL_IBR")
        binding_wsdl (str): Nombre del binding (ej: "CONSULTAR_INFO_GENERAL_IBR_Binding")
        namespace_wsdl (str): Namespace del WSDL (ej: "http://xmlns.bancocajasocial.com/co/servicios/abc/consultarInfoGeneralIbr/v1.0")

    Retorna:
        str: XML generado para el pipeline
    """
    
    wsdl_ref = wsdl_ref.replace("\\", "/")
    #service_target_ref = service_target_ref.replace("\\", "/")
    
    pipeline_xml = f"""<?xml version="1.0" encoding="UTF-8"?>
    <con:pipelineEntry xmlns:con="http://www.bea.com/wli/sb/pipeline/config" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
        <con:coreEntry>
            <con:binding type="SOAP" isSoap12="false" xsi:type="con:SoapBindingType">
                <con:wsdl ref="{wsdl_ref}"/>
                <con:binding>
                    <con:name>{binding_wsdl}</con:name>
                    <con:namespace>{namespace_wsdl}</con:namespace>
                </con:binding>
            </con:binding>
            <con:xqConfiguration>
                <con:snippetVersion>1.0</con:snippetVersion>
            </con:xqConfiguration>
        </con:coreEntry>
        <con:router errorHandler="error-ad486dc.N127d0eee.0.172040c1caf.N7dce">
            <con:pipeline name="error-ad486dc.N127d0eee.0.172040c1caf.N7dce" type="error">
                <con:stage name="stg_transformarMensajeError">
                    <con:context xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config"/>
                    <con:actions>
                        <con4:replace varName="manejarErrorRequest" contents-only="false" xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                            <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7cfe</con1:id>
                            <con2:expr>
                                <con5:xqueryTransform xmlns:con5="http://www.bea.com/wli/sb/stages/config">
                                    <con5:resource ref="ComponentesComunes/Resources/XQUERYs/xq_operacion_to_manejarError"/>
                                    <con5:param name="tns">
                                        <con5:path>data($tns)</con5:path>
                                    </con5:param>
                                    <con5:param name="fault">
                                        <con5:path>$fault/*</con5:path>
                                    </con5:param>
                                    <con5:param name="prefijo">
                                        <con5:path>data($prefijo)</con5:path>
                                    </con5:param>
                                    <con5:param name="operacion">
                                        <con5:path>data($operation)</con5:path>
                                    </con5:param>
                                    <con5:param name="body">
                                        <con5:path>$body/*</con5:path>
                                    </con5:param>
                                </con5:xqueryTransform>
                            </con2:expr>
                        </con4:replace>
                        <con4:wsCallout xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                            <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7cfd</con1:id>
                            <con2:service ref="ComponentesComunes/Proxies/PS_ManejadorGenericoErroresV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                            <con2:operation>manejarError</con2:operation>
                            <con2:request>
                                <con2:body wrapped="false">manejarErrorRequest</con2:body>
                            </con2:request>
                            <con2:response>
                                <con2:body wrapped="false">manejarErrorResponse</con2:body>
                            </con2:response>
                            <con2:requestTransform/>
                            <con2:responseTransform/>
                        </con4:wsCallout>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_transformarSalida">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config"/>
                    <con:actions>
                        <con2:replace contents-only="true" varName="body" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                            <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7c63</con1:id>
                            <con2:expr>
                                <con1:xqueryText xmlns:con5="http://www.bea.com/wli/sb/stages/config">$manejarErrorResponse</con1:xqueryText>
                            </con2:expr>
                        </con2:replace>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_transformacionBody">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0"/>
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearCuentaPasivo/v1.0"/>
                    </con:context>
                    <con:actions>
                        <con5:replace varName="transformacionBody" contents-only="false" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                            <con2:id>_ActionId-ad486cf.508f513.0.17624fa7dcd.N6529</con2:id>
                            <con1:expr>
                                <con2:xqueryTransform>
                                    <con2:resource ref="ComponentesComunes/Resources/XQUERYs/xq_Auditoria_to_RegistrarAuditoriaSOA"/>
                                    <con2:param name="codigoError">
                                        <con2:path>if(data($body/*:cabeceraSalida/*:tipoRespuesta) = "OK")
        then ("00") 
        else if (data($body/*:cabeceraSalida/*:respuestaError/*:codigoError)) then
        data($body/*:cabeceraSalida/*:respuestaError/*:codigoError)
        else
        data($fault/ctx:errorCode)</con2:path>
                                    </con2:param>
                                    <con2:param name="horaInicialTX">
                                        <con2:path>$horaInicialTx</con2:path>
                                    </con2:param>
                                    <con2:param name="mensajeResponse">
                                        <con2:path>$manejarErrorResponse</con2:path>
                                    </con2:param>
                                    <con2:param name="mensajeRequest">
                                        <con2:path>$mensajeEntradaABC</con2:path>
                                    </con2:param>
                                    <con2:param name="pId">
                                        <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:procesoId and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:procesoId)!='') then
          fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:procesoId)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="nombreFlujo">
                                        <con2:path>fn:concat("","ABC ",$operation)</con2:path>
                                    </con2:param>
                                    <con2:param name="archivoResponse">
                                        <con2:path>""</con2:path>
                                    </con2:param>
                                    <con2:param name="archivoRequest">
                                        <con2:path>""</con2:path>
                                    </con2:param>
                                    <con2:param name="oficina">
                                        <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoOficina and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoOficina)!='') then
          data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoOficina)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="numeroReferencia">
                                        <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)!='') then
          data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="tipoError">
                                        <con2:path>if ($manejarErrorResponse/*:cabeceraSalida/*:respuestaError/*:tipoError and fn:data($manejarErrorResponse/*:cabeceraSalida/*:respuestaError/*:tipoError)!='') then
          data($manejarErrorResponse/*:cabeceraSalida/*:respuestaError/*:tipoError)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="usuario">
                                        <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:usuario and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:usuario)!='') then
          data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:usuario)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="id">
                                        <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:identificadorTx and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:identificadorTx)!='') then
          data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:identificadorTx)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="descripcionError">
                                        <con2:path>data($fault/ctx:reason)</con2:path>
                                    </con2:param>
                                    <con2:param name="infoAdicional3">
                                        <con2:path>fn:concat("CANAL: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:canalOrigen)," SUBCANAL: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:subcanal))</con2:path>
                                    </con2:param>
                                    <con2:param name="descripcionTipoError">
                                        <con2:path>" "</con2:path>
                                    </con2:param>
                                    <con2:param name="infoAdicional2">
                                        <con2:path>fn:concat("NUMEROSOLICITUD: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud))</con2:path>
                                    </con2:param>
                                    <con2:param name="infoAdicional1">
                                        <con2:path>fn:concat("CODIGOCLIENTE: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoCliente))</con2:path>
                                    </con2:param>
                                    <con2:param name="codigoRespuesta">
                                        <con2:path>if ($manejarErrorResponse/*:cabeceraSalida/*:tipoRespuesta and fn:data($manejarErrorResponse/*:cabeceraSalida/*:tipoRespuesta)!='') then
          data($manejarErrorResponse/*:cabeceraSalida/*:tipoRespuesta)
        else
        data($fault/ctx:errorCode)</con2:path>
                                    </con2:param>
                                </con2:xqueryTransform>
                            </con1:expr>
                        </con5:replace>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_auditarMensaje">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cliente/V1.0"/>
                        <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Solicitud/V1.0"/>
                        <con2:userNsDecl prefix="v13" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Persona/V1.0"/>
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0"/>
                    </con:context>
                    <con:actions>
                        <con4:route xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad486cf.508f513.0.17624fa7dcd.N6531</con2:id>
                            <con4:service ref="UtilitariosEBS/Proxies/AuditoriaSOA/RegistrarAuditoriaSOADATV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                            <con4:operation>registrarAuditoria</con4:operation>
                            <con4:outboundTransform>
                                <con5:replace contents-only="true" varName="body" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                                    <con2:id>_ActionId-ad486cf.508f513.0.17624fa7dcd.N6530</con2:id>
                                    <con5:expr>
                                        <con2:xqueryText>$transformacionBody</con2:xqueryText>
                                    </con5:expr>
                                </con5:replace>
                            </con4:outboundTransform>
                        </con4:route>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_responder">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config"/>
                    <con:actions>
                        <con1:reply xmlns:con4="http://www.bea.com/wli/sb/stages/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                            <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7bf7</con1:id>
                        </con1:reply>
                    </con:actions>
                </con:stage>
            </con:pipeline>
            <con:pipeline name="request-ad486dc.N127d0eee.0.172040c1caf.N7b8f" type="request">
                <con:stage name="stg_inicializarVariables" id="_StageId-ad48653.391527db.0.1954e682498.N76db">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/typesystem/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config"/>
                    <con:actions>
                        <con3:assign varName="tns" xmlns:con1="http://www.bea.com/wli/sb/typesystem/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d9</con2:id>
                            <con3:expr>
                                <con2:xqueryText>fn:namespace-uri($body/*)</con2:xqueryText>
                            </con3:expr>
                        </con3:assign>
                        <con3:assign varName="prefijo" xmlns:con1="http://www.bea.com/wli/sb/typesystem/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d8</con2:id>
                            <con3:expr>
                                <con2:xqueryText>"srvabc"</con2:xqueryText>
                            </con3:expr>
                        </con3:assign>
                        <con1:assign varName="operacionABC" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d7</con2:id>
                            <con1:expr>
                                <con2:xqueryText>&lt;operacionABC>{{$operation}}&lt;/operacionABC></con2:xqueryText>
                            </con1:expr>
                        </con1:assign>
                        <con5:assign varName="horaInicialTx" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d6</con2:id>
                            <con5:expr>
                                <con2:xqueryText>fn:replace(fn:string( fn:current-dateTime()) , "T" ," ")</con2:xqueryText>
                            </con5:expr>
                        </con5:assign>
                        <con3:assign varName="mensajeEntradaAbc" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d5</con2:id>
                            <con3:expr>
                                <con2:xqueryText>$body</con2:xqueryText>
                            </con3:expr>
                        </con3:assign>
                        <con2:assign varName="mensajeEntradaCompleto" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config">
                            <con1:id>_ActionId-ad48653.391527db.0.1954e682498.N76d4</con1:id>
                            <con2:expr>
                                <con1:xqueryText><![CDATA[<Request>
            <MensajeEntradaABC>
            
            </MensajeEntradaABC>
            <MensajeEntradaLegado>
            
            </MensajeEntradaLegado>
        </Request>]]></con1:xqueryText>
                            </con2:expr>
                        </con2:assign>
                        <con2:assign varName="mensajeSalidaCompleto" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config">
                            <con1:id>_ActionId-ad48653.391527db.0.1954e682498.N76d3</con1:id>
                            <con2:expr>
                                <con1:xqueryText><![CDATA[<Response>
            <MensajeSalidaABC>
            
            </MensajeSalidaABC>
            <MensajeSalidaLegado>
            
            </MensajeSalidaLegado>
        </Response>]]></con1:xqueryText>
                            </con2:expr>
                        </con2:assign>
                        <con3:assign varName="operacionABCnode" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d2</con2:id>
                            <con3:expr>
                                <con2:xqueryText>fn:substring-after(fn:replace(fn:string(fn:node-name($body/*)),'Request',''),':')</con2:xqueryText>
                            </con3:expr>
                        </con3:assign>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_respaldarEntradaComun">
                    <con:context>
                        <con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInfoGeneralIbr/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/config"/>
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config"/>
                    </con:context>
                    <con:actions>
                        <con3:assign varName="mensajeEntradaABC" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad486e3.73c58b67.0.1720fe3b9a5.N7dac</con2:id>
                            <con3:expr>
                                <con2:xqueryText>$body/*</con2:xqueryText>
                            </con3:expr>
                        </con3:assign>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_transformacionEntrada">
                    <con:context>
                        <con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInfoGeneralIbr/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/config"/>
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarCarpeta/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config"/>
                    </con:context>
                    <con:actions>
                        <con3:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7964</con2:id>
                            <con3:expr>
                                <con2:xqueryText>$body</con2:xqueryText>
                            </con3:expr>
                        </con3:replace>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_respaldarEntradaLegado">
                    <con:context xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0"/>
                    </con:context>
                    <con:actions>
                        <con3:assign varName="mensajeEntradaABCLegado" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad486e3.73c58b67.0.1720fe3b9a5.N7d44</con2:id>
                            <con3:expr>
                                <con2:xqueryText>$body</con2:xqueryText>
                            </con3:expr>
                        </con3:assign>
                        <con1:insert varName="mensajeEntradaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N778a</con2:id>
                            <con1:location>
                                <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeEntradaABC</con2:xpathText>
                            </con1:location>
                            <con1:where>first-child</con1:where>
                            <con1:expr>
                                <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeEntradaABC</con2:xqueryText>
                            </con1:expr>
                        </con1:insert>
                        <con1:insert varName="mensajeEntradaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N76cf</con2:id>
                            <con1:location>
                                <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeEntradaLegado</con2:xpathText>
                            </con1:location>
                            <con1:where>first-child</con1:where>
                            <con1:expr>
                                <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeEntradaABCLegado</con2:xqueryText>
                            </con1:expr>
                        </con1:insert>
                    </con:actions>
                </con:stage>
            </con:pipeline>
            <con:pipeline name="response-ad486dc.N127d0eee.0.172040c1caf.N7b8e" type="response">
                <con:stage name="stg_transformacionSalida">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarCarpeta/v1.0"/>
                        <con2:varNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInfoGeneralIbr/v1.0"/>
                    </con:context>
                    <con:actions>
                        <con5:assign varName="mensajeSalidaABCLegado" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N74f8</con2:id>
                            <con3:expr>
                                <con:xqueryText xmlns:con="http://www.bea.com/wli/sb/stages/config">$body/*</con:xqueryText>
                            </con3:expr>
                        </con5:assign>
                        <con3:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7b22</con2:id>
                            <con3:expr>
                                <con2:xqueryText>$body</con2:xqueryText>
                            </con3:expr>
                        </con3:replace>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_transformacionBody" id="_StageId-ad48653.391527db.0.1954e682498.N75fe">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con2:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/servicios/AperturaEncFiduciario/v1.0"/>
                        <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0"/>
                        <con2:userNsDecl prefix="v14" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/AperturaEncFiduciario/v1.0"/>
                        <con2:userNsDecl prefix="v13" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/entidades/detalleFiduciaria/v1.0"/>
                        <con2:userNsDecl prefix="v16" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/abc/AperturaEncargoFid/v1.0"/>
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearCuentaPasivo/v1.0"/>
                        <con2:userNsDecl prefix="v15" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/entidades/Cabeceras/v1.0"/>
                    </con:context>
                    <con:actions>
                        <con5:assign varName="mensajeSalidaABC" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N75fd</con2:id>
                            <con3:expr>
                                <con2:xqueryText>$body/*</con2:xqueryText>
                            </con3:expr>
                        </con5:assign>
                        <con1:insert varName="mensajeSalidaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N75fc</con2:id>
                            <con1:location>
                                <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeSalidaABC</con2:xpathText>
                            </con1:location>
                            <con1:where>first-child</con1:where>
                            <con1:expr>
                                <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeSalidaABC</con2:xqueryText>
                            </con1:expr>
                        </con1:insert>
                        <con1:insert varName="mensajeSalidaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N75fb</con2:id>
                            <con1:location>
                                <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeSalidaLegado</con2:xpathText>
                            </con1:location>
                            <con1:where>first-child</con1:where>
                            <con1:expr>
                                <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeSalidaABCLegado</con2:xqueryText>
                            </con1:expr>
                        </con1:insert>
                        <con1:replace varName="transformacionBody" contents-only="false" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N75fa</con2:id>
                            <con1:expr>
                                <con2:xqueryTransform>
                                    <con2:resource ref="ComponentesComunes/Resources/XQUERYs/xq_Auditoria_to_RegistrarAuditoriaSOA"/>
                                    <con2:param name="codigoError">
                                        <con2:path>if ($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError and fn:data($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError)!='') then
          fn:data($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError)
        else('-')</con2:path>
                                    </con2:param>
                                    <con2:param name="horaInicialTX">
                                        <con2:path>$horaInicialTx</con2:path>
                                    </con2:param>
                                    <con2:param name="mensajeResponse">
                                        <con2:path>$mensajeSalidaCompleto</con2:path>
                                    </con2:param>
                                    <con2:param name="mensajeRequest">
                                        <con2:path>$mensajeEntradaCompleto</con2:path>
                                    </con2:param>
                                    <con2:param name="pId">
                                        <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:procesoId and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:procesoId)!='') then
          fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:procesoId)
        else('-')</con2:path>
                                    </con2:param>
                                    <con2:param name="nombreFlujo">
                                        <con2:path>fn:concat($operacionABC,"ABC")</con2:path>
                                    </con2:param>
                                    <con2:param name="archivoResponse">
                                        <con2:path>" "</con2:path>
                                    </con2:param>
                                    <con2:param name="archivoRequest">
                                        <con2:path>" "</con2:path>
                                    </con2:param>
                                    <con2:param name="oficina">
                                        <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:codigoOficina and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:codigoOficina)!='') then
          fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:codigoOficina)
        else('-')</con2:path>
                                    </con2:param>
                                    <con2:param name="numeroReferencia">
                                        <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:numeroSolicitud and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)!='') then
          fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)
        else(' ')</con2:path>
                                    </con2:param>
                                    <con2:param name="tipoError">
                                        <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:tipoError)</con2:path>
                                    </con2:param>
                                    <con2:param name="usuario">
                                        <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:usuario and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:usuario)!='') then
          fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:usuario)
        else('-')</con2:path>
                                    </con2:param>
                                    <con2:param name="id">
                                        <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:identificadorTx and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:identificadorTx)!='') then
          fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:identificadorTx)
        else('-')</con2:path>
                                    </con2:param>
                                    <con2:param name="descripcionError">
                                        <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:descripcionError)</con2:path>
                                    </con2:param>
                                    <con2:param name="infoAdicional3">
                                        <con2:path>" "</con2:path>
                                    </con2:param>
                                    <con2:param name="descripcionTipoError">
                                        <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:descripcionError)</con2:path>
                                    </con2:param>
                                    <con2:param name="infoAdicional2">
                                        <con2:path>fn:concat("CANAL: ",data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:canalOrigen), " | SUBCANAL: ", data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:subcanal))</con2:path>
                                    </con2:param>
                                    <con2:param name="infoAdicional1">
                                        <con2:path>fn:concat("NOMBREPROCESO :", data($copiaBody/*/*:encabezadoSolicitud/v13:nombreProceso), 
                  " | CODIGOTRANSACCION :", data($copiaBody/*/*:encabezadoSolicitud/*:codigoTransaccion),
                  " | NEMONICO :", data($copiaBody/*/*:detalleSolicitudApertura/*:tipoIdentificacion1), data($copiaBody/*/*:detalleSolicitudApertura/*:numeroIdentificacion1))</con2:path>
                                    </con2:param>
                                    <con2:param name="codigoRespuesta">
                                        <con2:path>data($body/*/*:cabeceraSalida/*:tipoRespuesta)</con2:path>
                                    </con2:param>
                                </con2:xqueryTransform>
                            </con1:expr>
                        </con1:replace>
                    </con:actions>
                </con:stage>
                <con:stage name="stg_auditarMensaje" id="_StageId-ad48653.391527db.0.1954e682498.N7590">
                    <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cliente/V1.0"/>
                        <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Solicitud/V1.0"/>
                        <con2:userNsDecl prefix="v13" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Persona/V1.0"/>
                        <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0"/>
                    </con:context>
                    <con:actions>
                        <con4:route xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                            <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N758f</con2:id>
                            <con4:service ref="UtilitariosEBS/Proxies/AuditoriaSOA/RegistrarAuditoriaSOADATV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                            <con4:operation>registrarAuditoria</con4:operation>
                            <con4:outboundTransform>
                                <con5:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                                    <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N758e</con2:id>
                                    <con1:expr>
                                        <con2:xqueryText>$transformacionBody</con2:xqueryText>
                                    </con1:expr>
                                </con5:replace>
                            </con4:outboundTransform>
                        </con4:route>
                    </con:actions>
                </con:stage>
            </con:pipeline>
            <con:flow>
                <con:pipeline-node name="PNN_flujoOperacion_JCA">
                    <con:request>request-ad486dc.N127d0eee.0.172040c1caf.N7b8f</con:request>
                    <con:response>response-ad486dc.N127d0eee.0.172040c1caf.N7b8e</con:response>
                </con:pipeline-node>
                <con:route-node name="RN_TO_SERVICIOS_BACKEND">
                    <con:context/>
                    <con:actions/>
                </con:route-node>
            </con:flow>
        </con:router>
    </con:pipelineEntry>"""
    return pipeline_xml

def generar_nombrado_abc(nombre, tipo="proxy", version="V1.0"):
    import re
    # 1. Convertir CamelCase → SNAKE_CASE
    snake = re.sub(r'(?<!^)(?=[A-Z])', '_', nombre).upper().strip()
    extension = ""
    
    # 2. Prefijo según tipo
    if tipo.lower() == "proxy":
        prefijo = "PS_"
        extension = ".proxy"
    elif tipo.lower() == "pipeline":
        prefijo = "PL_"
        extension = ".pipeline"
    elif tipo.lower() == "wsdl":
        prefijo = ""
        extension = ".wsdl"
    elif tipo.lower() == "nombre":
        prefijo = ""
    else:
        raise ValueError("Tipo no reconocido. Usa 'proxy', 'pipeline', 'wsdl' o 'nombre'.")
        
    return f"{prefijo}{snake}DA{version}{extension}"

def generar_nombrado_ebs(nombre, tipo="proxy", version="V1.0"):
    extension = ""

    # 1️⃣ Determinar extensión según tipo
    tipo = tipo.lower()
    if tipo == "proxy":
        extension = ".proxy"
    elif tipo == "pipeline":
        extension = ".pipeline"
    elif tipo == "wsdl":
        extension = ".wsdl"
    elif tipo == "nombre":
        extension = ""
    else:
        raise ValueError("Tipo no reconocido. Usa 'proxy', 'pipeline', 'wsdl' o 'nombre'.")

    # 2️⃣ Tomar el nombre base
    base = st.session_state["service_name_ebs"]

    # 3️⃣ Asegurar que la primera letra esté en mayúscula y conservar el resto (camelCase)
    base = base[0].upper() + base[1:]

    # 4️⃣ Asegurar que si contiene 'ASV2.1', quede en mayúsculas
    #base = base.replace("Asv2.1", "ASV2.1").replace("asv2.1", "ASV2.1")

    return f"{base}{extension}"

def crear_wsdl_ebs(operation_name: str,
                   wsdl_path: str,
                   xsd_path: str,
                   target_namespace_xsd: str,
                   ns_elem_prefix: str,
                   write_to_file: bool = False) -> str:
    """
    Genera un WSDL para la capa ABC (siempre con versión v1.0).

    Args:
        operation_name: Nombre de la operación (ej: "consultarInfoArchivoIngresoPrestamo").
        wsdl_path: Ruta absoluta al archivo WSDL de salida.
        xsd_path: Ruta relativa o absoluta del archivo XSD.
        target_namespace_xsd: Namespace del XSD.
        ns_elem_prefix: Prefijo para el namespace del XSD (ej: "serconsinfarchingrpres").
        write_to_file: Si True, escribe el archivo en disco.

    Returns:
        str: Contenido del WSDL generado en formato string.
    """
    import xml.etree.ElementTree as ET
    import os
    from collections import OrderedDict
    from pathlib import PurePosixPath

    # URIs
    WSDL_URI = "http://schemas.xmlsoap.org/wsdl/"
    SOAP11_URI = "http://schemas.xmlsoap.org/wsdl/soap/"
    SOAP12_URI = "http://schemas.xmlsoap.org/wsdl/soap12/"
    MIME_URI = "http://schemas.xmlsoap.org/wsdl/mime/"
    XSD_URI = "http://www.w3.org/2001/XMLSchema"

    version = "v1.0"  # fijo
    tns_wsdl = f"http://xmlns.bancocajasocial.com/co/servicios/ebs/{operation_name}/{version}"

    # Calcular schemaLocation relativo desde wsdl_path hasta xsd_path (siempre con /)
    schema_location = os.path.relpath(
        xsd_path,
        start=os.path.dirname(wsdl_path)
    ).replace("\\", "/")

    # Namespaces (sin duplicar soap)
    ET.register_namespace("", WSDL_URI)
    ET.register_namespace("soap", SOAP11_URI)
    ET.register_namespace("xsd", XSD_URI)

    # Definiciones
    defs_attribs = OrderedDict()
    defs_attribs["targetNamespace"] = tns_wsdl
    defs_attribs["xmlns"] = WSDL_URI
    defs_attribs["xmlns:tns"] = tns_wsdl
    defs_attribs["xmlns:soap12"] = SOAP12_URI
    defs_attribs["xmlns:mime"] = MIME_URI
    defs_attribs[f"xmlns:{ns_elem_prefix}"] = target_namespace_xsd

    definitions = ET.Element("definitions", attrib=defs_attribs)

    # <types> con schema targetNamespace y schema import
    types = ET.SubElement(definitions, "types")
    ET.SubElement(types, f"{{{XSD_URI}}}schema", {
        "targetNamespace": f"{tns_wsdl}/types",
        "elementFormDefault": "qualified"
    })
    schema_import_block = ET.SubElement(types, f"{{{XSD_URI}}}schema")
    ET.SubElement(schema_import_block, f"{{{XSD_URI}}}import", {
        "schemaLocation": schema_location,
        "namespace": target_namespace_xsd
    })

    # mensajes
    msg_in = ET.SubElement(definitions, "message", {"name": f"{operation_name}Request"})
    ET.SubElement(msg_in, "part", {
        "name": f"{operation_name}Request",
        "element": f"{ns_elem_prefix}:{operation_name}Request"
    })

    msg_out = ET.SubElement(definitions, "message", {"name": f"{operation_name}Response"})
    ET.SubElement(msg_out, "part", {
        "name": f"{operation_name}Response",
        "element": f"{ns_elem_prefix}:{operation_name}Response"
    })

    # portType
    port_type_name = f"{to_upper_snake_case(operation_name)}_PORT"
    portType = ET.SubElement(definitions, "portType", {"name": port_type_name})
    op = ET.SubElement(portType, "operation", {"name": operation_name})
    ET.SubElement(op, "input", {"message": f"tns:{operation_name}Request"})
    ET.SubElement(op, "output", {"message": f"tns:{operation_name}Response"})

    # binding
    binding_name = f"{to_upper_snake_case(operation_name)}_Binding"
    binding = ET.SubElement(definitions, "binding", {"name": binding_name, "type": f"tns:{port_type_name}"})
    ET.SubElement(binding, f"{{{SOAP11_URI}}}binding", {
        "style": "document",
        "transport": "http://schemas.xmlsoap.org/soap/http"
    })
    bop = ET.SubElement(binding, "operation", {"name": operation_name})
    # Ajuste del soapAction: SOLO hasta /v1.0
    ET.SubElement(bop, f"{{{SOAP11_URI}}}operation", {
        "style": "document",
        "soapAction": f"{tns_wsdl}"
    })
    binp = ET.SubElement(bop, "input")
    ET.SubElement(binp, f"{{{SOAP11_URI}}}body", {
        "use": "literal", "parts": f"{operation_name}Request"
    })
    bout = ET.SubElement(bop, "output")
    ET.SubElement(bout, f"{{{SOAP11_URI}}}body", {
        "use": "literal", "parts": f"{operation_name}Response"
    })

    raw_str = ET.tostring(definitions, encoding="unicode")
    wsdl_str = prettify(raw_str)

    # Guardar en archivo
    if write_to_file:
        dirpath = os.path.dirname(wsdl_path)
        if dirpath and not os.path.exists(dirpath):
            os.makedirs(dirpath, exist_ok=True)
        ET.ElementTree(definitions).write(wsdl_path, encoding="utf-8", xml_declaration=True)

    return wsdl_str

def crear_proxy_ebs(wsdl_ref: str, binding_wsdl: str, namespace_wsdl: str, pipeline_ref: str) -> str:
    """
    Crea el XML de un proxy service parametrizado con el wsdl, binding, namespace y pipeline.

    Parámetros:
        wsdl_ref (str): Ruta del WSDL en OSB (ej: "STARLT_ABC/Resources/WSDLs/CONSULTAR_INFO_GENERAL_IBR")
        binding_wsdl (str): Nombre del binding (ej: "CONSULTAR_INFO_GENERAL_IBR_Binding")
        namespace_wsdl (str): Namespace del WSDL (ej: "http://xmlns.bancocajasocial.com/co/servicios/abc/consultarInfoGeneralIbr/v1.0")
        pipeline_ref (str): Ruta del pipeline en OSB (ej: "STARLT_ABC/Pipeline/PL_CONSULTAR_INFO_GENERAL_IBRDAV1.0")

    Retorna:
        str: XML generado para el proxy
    """
    
    # 🔑 Normalizar paths a formato OSB (/ en vez de \)
    wsdl_ref = wsdl_ref.replace("\\", "/")
    pipeline_ref = pipeline_ref.replace("\\", "/")

    proxy_xml = f"""<?xml version="1.0" encoding="UTF-8"?>
    <ser:proxyServiceEntry xmlns:ser="http://www.bea.com/wli/sb/services" xmlns:con="http://www.bea.com/wli/sb/services/security/config" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:oper="http://xmlns.oracle.com/servicebus/proxy/operations" xmlns:tran="http://www.bea.com/wli/sb/transports">
        <ser:coreEntry>
            <ser:security>
                <con:inboundWss processWssHeader="true"/>
            </ser:security>
            <ser:binding type="SOAP" xsi:type="con:SoapBindingType" isSoap12="false" xmlns:con="http://www.bea.com/wli/sb/services/bindings/config">
                <con:wsdl ref="{wsdl_ref}"/>
                <con:binding>
                    <con:name>{binding_wsdl}</con:name>
                    <con:namespace>{namespace_wsdl}</con:namespace>
                </con:binding>
                <con:selector type="SOAP body"/>
            </ser:binding>
            <oper:operations enabled="true"/>
            <ser:ws-policy>
                <ser:binding-mode>no-policies</ser:binding-mode>
            </ser:ws-policy>
            <ser:invoke ref="{pipeline_ref}" xsi:type="con:PipelineRef" xmlns:con="http://www.bea.com/wli/sb/pipeline/config"/>
            <ser:xqConfiguration>
                <ser:snippetVersion>1.0</ser:snippetVersion>
            </ser:xqConfiguration>
        </ser:coreEntry>
        <ser:endpointConfig>
            <tran:provider-id>local</tran:provider-id>
            <tran:inbound>true</tran:inbound>
            <tran:inbound-properties/>
        </ser:endpointConfig>
    </ser:proxyServiceEntry>"""
    return proxy_xml

def crear_pipeline_ebs(wsdl_ref: str, binding_wsdl: str, namespace_wsdl: str, operation_name: str) -> str:
    """
    Crea el XML de un pipeline parametrizado con el wsdl, binding y namespace.

    Parámetros:
        wsdl_ref (str): Ruta del WSDL en OSB (ej: "STARLT_ABC/Resources/WSDLs/CONSULTAR_INFO_GENERAL_IBR")
        binding_wsdl (str): Nombre del binding (ej: "CONSULTAR_INFO_GENERAL_IBR_Binding")
        namespace_wsdl (str): Namespace del WSDL (ej: "http://xmlns.bancocajasocial.com/co/servicios/abc/consultarInfoGeneralIbr/v1.0")

    Retorna:
        str: XML generado para el pipeline
    """
    
    wsdl_ref = wsdl_ref.replace("\\", "/")
    #service_target_ref = service_target_ref.replace("\\", "/")
    
    pipeline_xml = f"""<?xml version="1.0" encoding="UTF-8"?>
    <con:pipelineEntry xmlns:con="http://www.bea.com/wli/sb/pipeline/config" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
        <con:coreEntry>
            <con:binding type="SOAP" isSoap12="false" xsi:type="con:SoapBindingType">
                <con:wsdl ref="{wsdl_ref}"/>
                <con:binding>
                    <con:name>{binding_wsdl}</con:name>
                    <con:namespace>{namespace_wsdl}</con:namespace>
                </con:binding>
            </con:binding>
            <con:xqConfiguration>
                <con:snippetVersion>1.0</con:snippetVersion>
            </con:xqConfiguration>
        </con:coreEntry>
        <con:router errorHandler="error-ad486dc.N127d0eee.0.172040c1caf.N7dce">
        <con:pipeline name="error-ad486dc.N127d0eee.0.172040c1caf.N7dce" type="error">
            <con:stage name="stg_transformarMensajeError">
                <con:context xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config"/>
                <con:actions>
                    <con4:replace varName="manejarErrorRequest" contents-only="false" xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7cfe</con1:id>
                        <con2:expr>
                            <con5:xqueryTransform xmlns:con5="http://www.bea.com/wli/sb/stages/config">
                                <con5:resource ref="ComponentesComunes/Resources/XQUERYs/xq_operacion_to_manejarError"/>
                                <con5:param name="tns">
                                    <con5:path>data($espacioNombreABC)</con5:path>
                                </con5:param>
                                <con5:param name="fault">
                                    <con5:path>$fault/*</con5:path>
                                </con5:param>
                                <con5:param name="prefijo">
                                    <con5:path>data($prefijoABC)</con5:path>
                                </con5:param>
                                <con5:param name="operacion">
                                    <con5:path>data($operation)</con5:path>
                                </con5:param>
                                <con5:param name="body">
                                    <con5:path>$body/*</con5:path>
                                </con5:param>
                            </con5:xqueryTransform>
                        </con2:expr>
                    </con4:replace>
                    <con4:wsCallout xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7cfd</con1:id>
                        <con2:service ref="ComponentesComunes/Proxies/PS_ManejadorGenericoErroresV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                        <con2:operation>manejarError</con2:operation>
                        <con2:request>
                            <con2:body wrapped="false">manejarErrorRequest</con2:body>
                        </con2:request>
                        <con2:response>
                            <con2:body wrapped="false">manejarErrorResponse</con2:body>
                        </con2:response>
                        <con2:requestTransform/>
                        <con2:responseTransform/>
                    </con4:wsCallout>
                </con:actions>
            </con:stage>
            <con:stage name="stg_transformarSalida">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config"/>
                <con:actions>
                    <con2:replace contents-only="true" varName="body" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7c63</con1:id>
                        <con2:expr>
                            <con1:xqueryText xmlns:con5="http://www.bea.com/wli/sb/stages/config">$manejarErrorResponse</con1:xqueryText>
                        </con2:expr>
                    </con2:replace>
                </con:actions>
            </con:stage>
            <con:stage name="stg_transformacionBody">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                    <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0"/>
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearCuentaPasivo/v1.0"/>
                </con:context>
                <con:actions>
                    <con5:replace varName="transformacionBody" contents-only="false" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con2:id>_ActionId-ad486cf.508f513.0.17624fa7dcd.N6529</con2:id>
                        <con1:expr>
                            <con2:xqueryTransform>
                                <con2:resource ref="ComponentesComunes/Resources/XQUERYs/xq_Auditoria_to_RegistrarAuditoriaSOA"/>
                                <con2:param name="codigoError">
                                    <con2:path>if(data($body/*:cabeceraSalida/*:tipoRespuesta) = "OK")
    then ("00") 
    else data($body/*:cabeceraSalida/*:respuestaError/*:codigoError)</con2:path>
                                </con2:param>
                                <con2:param name="horaInicialTX">
                                    <con2:path>$horaInicialTx</con2:path>
                                </con2:param>
                                <con2:param name="mensajeResponse">
                                    <con2:path>$manejarErrorResponse</con2:path>
                                </con2:param>
                                <con2:param name="mensajeRequest">
                                    <con2:path>$mensajeEntradaABC</con2:path>
                                </con2:param>
                                <con2:param name="pId">
                                    <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:procesoId and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:procesoId)!='') then
      fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:procesoId)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="nombreFlujo">
                                    <con2:path>fn:concat("","ABC ",$operacionABC)</con2:path>
                                </con2:param>
                                <con2:param name="archivoResponse">
                                    <con2:path>""</con2:path>
                                </con2:param>
                                <con2:param name="archivoRequest">
                                    <con2:path>""</con2:path>
                                </con2:param>
                                <con2:param name="oficina">
                                    <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoOficina and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoOficina)!='') then
      data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoOficina)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="numeroReferencia">
                                    <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)!='') then
      data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="tipoError">
                                    <con2:path>if ($manejarErrorResponse/*:cabeceraSalida/*:respuestaError/*:tipoError and fn:data($manejarErrorResponse/*:cabeceraSalida/*:respuestaError/*:tipoError)!='') then
      data($manejarErrorResponse/*:cabeceraSalida/*:respuestaError/*:tipoError)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="usuario">
                                    <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:usuario and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:usuario)!='') then
      data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:usuario)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="id">
                                    <con2:path>if ($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:identificadorTx and fn:data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:identificadorTx)!='') then
      data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:identificadorTx)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="descripcionError">
                                    <con2:path>data($fault/*)</con2:path>
                                </con2:param>
                                <con2:param name="infoAdicional3">
                                    <con2:path>fn:concat("CANAL: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:canalOrigen)," SUBCANAL: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:subcanal))</con2:path>
                                </con2:param>
                                <con2:param name="descripcionTipoError">
                                    <con2:path>" "</con2:path>
                                </con2:param>
                                <con2:param name="infoAdicional2">
                                    <con2:path>fn:concat("NUMEROSOLICITUD: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:numeroSolicitud))</con2:path>
                                </con2:param>
                                <con2:param name="infoAdicional1">
                                    <con2:path>fn:concat("CODIGOCLIENTE: ",data($mensajeEntradaABC/*:cabeceraEntrada/*:invocador/*:codigoCliente))</con2:path>
                                </con2:param>
                                <con2:param name="codigoRespuesta">
                                    <con2:path>if ($manejarErrorResponse/*:cabeceraSalida/*:tipoRespuesta and fn:data($manejarErrorResponse/*:cabeceraSalida/*:tipoRespuesta)!='') then
      data($manejarErrorResponse/*:cabeceraSalida/*:tipoRespuesta)
    else(' ')</con2:path>
                                </con2:param>
                            </con2:xqueryTransform>
                        </con1:expr>
                    </con5:replace>
                </con:actions>
            </con:stage>
            <con:stage name="stg_auditarMensaje">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                    <con2:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cliente/V1.0"/>
                    <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Solicitud/V1.0"/>
                    <con2:userNsDecl prefix="v13" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Persona/V1.0"/>
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0"/>
                </con:context>
                <con:actions>
                    <con4:route xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad486cf.508f513.0.17624fa7dcd.N6531</con2:id>
                        <con4:service ref="UtilitariosEBS/Proxies/AuditoriaSOA/RegistrarAuditoriaSOADATV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                        <con4:operation>registrarAuditoria</con4:operation>
                        <con4:outboundTransform>
                            <con5:replace contents-only="true" varName="body" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                                <con2:id>_ActionId-ad486cf.508f513.0.17624fa7dcd.N6530</con2:id>
                                <con5:expr>
                                    <con2:xqueryText>$transformacionBody</con2:xqueryText>
                                </con5:expr>
                            </con5:replace>
                        </con4:outboundTransform>
                    </con4:route>
                </con:actions>
            </con:stage>
            <con:stage name="stg_responder">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config"/>
                <con:actions>
                    <con1:reply xmlns:con4="http://www.bea.com/wli/sb/stages/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con1:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7bf7</con1:id>
                    </con1:reply>
                </con:actions>
            </con:stage>
        </con:pipeline>
        <con:pipeline name="request-ad486dc.N127d0eee.0.172040c1caf.N7b8f" type="request">
            <con:stage name="stg_inicializarVariables" id="_StageId-ad48653.391527db.0.1954e682498.N76db">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/typesystem/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config"/>
                <con:actions>
                    <con3:assign varName="tns" xmlns:con1="http://www.bea.com/wli/sb/typesystem/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d9</con2:id>
                        <con3:expr>
                            <con2:xqueryText>fn:namespace-uri($body/*)</con2:xqueryText>
                        </con3:expr>
                    </con3:assign>
                    <con3:assign varName="prefijo" xmlns:con1="http://www.bea.com/wli/sb/typesystem/config" xmlns:con4="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/logging/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d8</con2:id>
                        <con3:expr>
                            <con2:xqueryText>"srvabc"</con2:xqueryText>
                        </con3:expr>
                    </con3:assign>
                    <con1:assign varName="operacionABC" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d7</con2:id>
                        <con1:expr>
                            <con2:xqueryText>&lt;operacionABC>{{$operation}}&lt;/operacionABC></con2:xqueryText>
                        </con1:expr>
                    </con1:assign>
                    <con5:assign varName="horaInicialTx" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d6</con2:id>
                        <con5:expr>
                            <con2:xqueryText>fn:replace(fn:string( fn:current-dateTime()) , "T" ," ")</con2:xqueryText>
                        </con5:expr>
                    </con5:assign>
                    <con3:assign varName="mensajeEntradaAbc" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d5</con2:id>
                        <con3:expr>
                            <con2:xqueryText>$body</con2:xqueryText>
                        </con3:expr>
                    </con3:assign>
                    <con2:assign varName="mensajeEntradaCompleto" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config">
                        <con1:id>_ActionId-ad48653.391527db.0.1954e682498.N76d4</con1:id>
                        <con2:expr>
                            <con1:xqueryText><![CDATA[<Request>
        <MensajeEntradaABC>
        
        </MensajeEntradaABC>
        <MensajeEntradaLegado>
        
        </MensajeEntradaLegado>
    </Request>]]></con1:xqueryText>
                        </con2:expr>
                    </con2:assign>
                    <con2:assign varName="mensajeSalidaCompleto" xmlns:con4="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con2="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con1="http://www.bea.com/wli/sb/stages/config">
                        <con1:id>_ActionId-ad48653.391527db.0.1954e682498.N76d3</con1:id>
                        <con2:expr>
                            <con1:xqueryText><![CDATA[<Response>
        <MensajeSalidaABC>
        
        </MensajeSalidaABC>
        <MensajeSalidaLegado>
        
        </MensajeSalidaLegado>
    </Response>]]></con1:xqueryText>
                        </con2:expr>
                    </con2:assign>
                    <con3:assign varName="operacionABCnode" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N76d2</con2:id>
                        <con3:expr>
                            <con2:xqueryText>fn:substring-after(fn:replace(fn:string(fn:node-name($body/*)),'Request',''),':')</con2:xqueryText>
                        </con3:expr>
                    </con3:assign>
                </con:actions>
            </con:stage>
            <con:stage name="stg_respaldarEntradaComun">
                <con:context>
                    <con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInfoGeneralIbr/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/config"/>
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config"/>
                </con:context>
                <con:actions>
                    <con3:assign varName="mensajeEntradaABC" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad486e3.73c58b67.0.1720fe3b9a5.N7dac</con2:id>
                        <con3:expr>
                            <con2:xqueryText>$body/*</con2:xqueryText>
                        </con3:expr>
                    </con3:assign>
                </con:actions>
            </con:stage>
            <con:stage name="stg_transformacionEntrada">
                <con:context>
                    <con1:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInfoGeneralIbr/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/config"/>
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarCarpeta/v1.0" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config"/>
                </con:context>
                <con:actions>
                    <con3:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7964</con2:id>
                        <con3:expr>
                            <con2:xqueryText>$body</con2:xqueryText>
                        </con3:expr>
                    </con3:replace>
                </con:actions>
            </con:stage>
            <con:stage name="stg_respaldarEntradaLegado">
                <con:context xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config">
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0"/>
                </con:context>
                <con:actions>
                    <con3:assign varName="mensajeEntradaABCLegado" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad486e3.73c58b67.0.1720fe3b9a5.N7d44</con2:id>
                        <con3:expr>
                            <con2:xqueryText>$body</con2:xqueryText>
                        </con3:expr>
                    </con3:assign>
                    <con1:insert varName="mensajeEntradaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N778a</con2:id>
                        <con1:location>
                            <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeEntradaABC</con2:xpathText>
                        </con1:location>
                        <con1:where>first-child</con1:where>
                        <con1:expr>
                            <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeEntradaABC</con2:xqueryText>
                        </con1:expr>
                    </con1:insert>
                    <con1:insert varName="mensajeEntradaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N76cf</con2:id>
                        <con1:location>
                            <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeEntradaLegado</con2:xpathText>
                        </con1:location>
                        <con1:where>first-child</con1:where>
                        <con1:expr>
                            <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeEntradaABCLegado</con2:xqueryText>
                        </con1:expr>
                    </con1:insert>
                </con:actions>
            </con:stage>
        </con:pipeline>
        <con:pipeline name="response-ad486dc.N127d0eee.0.172040c1caf.N7b8e" type="response">
            <con:stage name="stg_transformacionSalida">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarCarpeta/v1.0"/>
                    <con2:varNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/consultarInfoGeneralIbr/v1.0"/>
                </con:context>
                <con:actions>
                    <con5:assign varName="mensajeSalidaABCLegado" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N74f8</con2:id>
                        <con3:expr>
                            <con:xqueryText xmlns:con="http://www.bea.com/wli/sb/stages/config">$body/*</con:xqueryText>
                        </con3:expr>
                    </con5:assign>
                    <con3:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad486dc.N127d0eee.0.172040c1caf.N7b22</con2:id>
                        <con3:expr>
                            <con2:xqueryText>$body</con2:xqueryText>
                        </con3:expr>
                    </con3:replace>
                </con:actions>
            </con:stage>
            <con:stage name="stg_transformacionBody" id="_StageId-ad48653.391527db.0.1954e682498.N75fe">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                    <con2:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/servicios/AperturaEncFiduciario/v1.0"/>
                    <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cabeceras/V1.0"/>
                    <con2:userNsDecl prefix="v14" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/AperturaEncFiduciario/v1.0"/>
                    <con2:userNsDecl prefix="v13" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/entidades/detalleFiduciaria/v1.0"/>
                    <con2:userNsDecl prefix="v16" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/abc/AperturaEncargoFid/v1.0"/>
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearCuentaPasivo/v1.0"/>
                    <con2:userNsDecl prefix="v15" namespace="http://xmlns.bancocajasocial.com/co/canales/schemas/entidades/Cabeceras/v1.0"/>
                </con:context>
                <con:actions>
                    <con5:assign varName="mensajeSalidaABC" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N75fd</con2:id>
                        <con3:expr>
                            <con2:xqueryText>$body/*</con2:xqueryText>
                        </con3:expr>
                    </con5:assign>
                    <con1:insert varName="mensajeSalidaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N75fc</con2:id>
                        <con1:location>
                            <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeSalidaABC</con2:xpathText>
                        </con1:location>
                        <con1:where>first-child</con1:where>
                        <con1:expr>
                            <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeSalidaABC</con2:xqueryText>
                        </con1:expr>
                    </con1:insert>
                    <con1:insert varName="mensajeSalidaCompleto" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id xmlns:con2="http://www.bea.com/wli/sb/stages/config">_ActionId-ad48653.391527db.0.1954e682498.N75fb</con2:id>
                        <con1:location>
                            <con2:xpathText xmlns:con2="http://www.bea.com/wli/sb/stages/config">./*:MensajeSalidaLegado</con2:xpathText>
                        </con1:location>
                        <con1:where>first-child</con1:where>
                        <con1:expr>
                            <con2:xqueryText xmlns:con2="http://www.bea.com/wli/sb/stages/config">$mensajeSalidaABCLegado</con2:xqueryText>
                        </con1:expr>
                    </con1:insert>
                    <con1:replace varName="transformacionBody" contents-only="false" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N75fa</con2:id>
                        <con1:expr>
                            <con2:xqueryTransform>
                                <con2:resource ref="ComponentesComunes/Resources/XQUERYs/xq_Auditoria_to_RegistrarAuditoriaSOA"/>
                                <con2:param name="codigoError">
                                    <con2:path>if ($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError and fn:data($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError)!='') then
      fn:data($body/*/*:cabeceraSalida/*:respuestaError/*:codigoError)
    else('-')</con2:path>
                                </con2:param>
                                <con2:param name="horaInicialTX">
                                    <con2:path>$horaInicialTx</con2:path>
                                </con2:param>
                                <con2:param name="mensajeResponse">
                                    <con2:path>$mensajeSalidaCompleto</con2:path>
                                </con2:param>
                                <con2:param name="mensajeRequest">
                                    <con2:path>$mensajeEntradaCompleto</con2:path>
                                </con2:param>
                                <con2:param name="pId">
                                    <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:procesoId and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:procesoId)!='') then
      fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:procesoId)
    else('-')</con2:path>
                                </con2:param>
                                <con2:param name="nombreFlujo">
                                    <con2:path>fn:concat($operacionABC,"ABC")</con2:path>
                                </con2:param>
                                <con2:param name="archivoResponse">
                                    <con2:path>" "</con2:path>
                                </con2:param>
                                <con2:param name="archivoRequest">
                                    <con2:path>" "</con2:path>
                                </con2:param>
                                <con2:param name="oficina">
                                    <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:codigoOficina and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:codigoOficina)!='') then
      fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:codigoOficina)
    else('-')</con2:path>
                                </con2:param>
                                <con2:param name="numeroReferencia">
                                    <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:numeroSolicitud and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)!='') then
      fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:numeroSolicitud)
    else(' ')</con2:path>
                                </con2:param>
                                <con2:param name="tipoError">
                                    <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:tipoError)</con2:path>
                                </con2:param>
                                <con2:param name="usuario">
                                    <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:usuario and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:usuario)!='') then
      fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:usuario)
    else('-')</con2:path>
                                </con2:param>
                                <con2:param name="id">
                                    <con2:path>if ($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:identificadorTx and fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:identificadorTx)!='') then
      fn:data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:identificadorTx)
    else('-')</con2:path>
                                </con2:param>
                                <con2:param name="descripcionError">
                                    <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:descripcionError)</con2:path>
                                </con2:param>
                                <con2:param name="infoAdicional3">
                                    <con2:path>" "</con2:path>
                                </con2:param>
                                <con2:param name="descripcionTipoError">
                                    <con2:path>data($body/*/*:cabeceraSalida/*:respuestaError/*:descripcionError)</con2:path>
                                </con2:param>
                                <con2:param name="infoAdicional2">
                                    <con2:path>fn:concat("CANAL: ",data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:canalOrigen), " | SUBCANAL: ", data($mensajeEntradaAbc/*/*:cabeceraEntrada/*:invocador/*:subcanal))</con2:path>
                                </con2:param>
                                <con2:param name="infoAdicional1">
                                    <con2:path>fn:concat("NOMBREPROCESO :", data($copiaBody/*/*:encabezadoSolicitud/v13:nombreProceso), 
              " | CODIGOTRANSACCION :", data($copiaBody/*/*:encabezadoSolicitud/*:codigoTransaccion),
              " | NEMONICO :", data($copiaBody/*/*:detalleSolicitudApertura/*:tipoIdentificacion1), data($copiaBody/*/*:detalleSolicitudApertura/*:numeroIdentificacion1))</con2:path>
                                </con2:param>
                                <con2:param name="codigoRespuesta">
                                    <con2:path>data($body/*/*:cabeceraSalida/*:tipoRespuesta)</con2:path>
                                </con2:param>
                            </con2:xqueryTransform>
                        </con1:expr>
                    </con1:replace>
                </con:actions>
            </con:stage>
            <con:stage name="stg_auditarMensaje" id="_StageId-ad48653.391527db.0.1954e682498.N7590">
                <con:context xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                    <con2:userNsDecl prefix="v12" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Cliente/V1.0"/>
                    <con2:userNsDecl prefix="v11" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Solicitud/V1.0"/>
                    <con2:userNsDecl prefix="v13" namespace="http://xmlns.bancocajasocial.com/co/comunes/schema/Persona/V1.0"/>
                    <con2:userNsDecl prefix="v1" namespace="http://xmlns.bancocajasocial.com/co/schemas/operacion/crearTarjetaCredito/v1.0"/>
                </con:context>
                <con:actions>
                    <con4:route xmlns:con1="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con4="http://www.bea.com/wli/sb/stages/publish/config" xmlns:con2="http://www.bea.com/wli/sb/stages/config" xmlns:con3="http://www.bea.com/wli/sb/stages/transform/config">
                        <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N758f</con2:id>
                        <con4:service ref="UtilitariosEBS/Proxies/AuditoriaSOA/RegistrarAuditoriaSOADATV1.0" xsi:type="ref:ProxyRef" xmlns:ref="http://www.bea.com/wli/sb/reference"/>
                        <con4:operation>registrarAuditoria</con4:operation>
                        <con4:outboundTransform>
                            <con5:replace varName="body" contents-only="true" xmlns:con1="http://www.bea.com/wli/sb/stages/transform/config" xmlns:con3="http://www.bea.com/wli/sb/stages/routing/config" xmlns:con5="http://www.bea.com/wli/sb/stages/transform/config">
                                <con2:id>_ActionId-ad48653.391527db.0.1954e682498.N758e</con2:id>
                                <con1:expr>
                                    <con2:xqueryText>$transformacionBody</con2:xqueryText>
                                </con1:expr>
                            </con5:replace>
                        </con4:outboundTransform>
                    </con4:route>
                </con:actions>
            </con:stage>
        </con:pipeline>
        <con:flow>
            <con:pipeline-node name="stg_EBS">
                <con:request>request-ad486dc.N127d0eee.0.172040c1caf.N7b8f</con:request>
                <con:response>response-ad486dc.N127d0eee.0.172040c1caf.N7b8e</con:response>
            </con:pipeline-node>
        </con:flow>
    </con:router>
    </con:pipelineEntry>"""
    return pipeline_xml

def generar_nombrado_exp(nombre, tipo="proxy"):
    # 1. Convertir CamelCase → SNAKE_CASE
    extension = ""
    
    # 2. Prefijo según tipo
    if tipo.lower() == "proxy":
        extension = ".proxy"
    elif tipo.lower() == "pipeline":
        extension = ".pipeline"
    elif tipo.lower() == "wsdl":
        extension = ".wsdl"
    else:
        raise ValueError("Tipo no reconocido. Usa 'proxy' o 'pipeline'.")
    
    # 3. Armar resultado con la versión pasada como parámetro
    return f"{nombre}{extension}"

def exportar_proyecto_zip():
    with tempfile.TemporaryDirectory() as tmpdir:
        # --- Crear estructura EXP ---
        exp_root = os.path.join(tmpdir, f"{st.session_state['exp_proyecto']}")
        
        if st.session_state["tipo_servicio"] == "Nuevo":
            proxy_exp_rel_path = st.session_state["ubicacion_proxy_exp"]
            proxy_exp_abs_path = os.path.join(tmpdir, proxy_exp_rel_path)  
            
            #Proxy EXP
            os.makedirs(os.path.dirname(proxy_exp_abs_path), exist_ok=True)
            with open(proxy_exp_abs_path, "w", encoding="utf-8") as f:
                f.write(st.session_state["archivo_proxy_exp"])
        
        
        pipeline_exp_rel_path = st.session_state["ubicacion_pipeline_exp"]
        pipeline_exp_abs_path = os.path.join(tmpdir, pipeline_exp_rel_path)
        wsdl_exp_rel_path = st.session_state["ubicacion_wsdl_exp"]
        wsdl_exp_abs_path = os.path.join(tmpdir, wsdl_exp_rel_path) 

            
        #Pipeline EXP
        os.makedirs(os.path.dirname(pipeline_exp_abs_path), exist_ok=True)
        with open(pipeline_exp_abs_path, "w", encoding="utf-8") as f:
            f.write(st.session_state["archivo_pipeline_exp"])
            
        #WSDL EXP
        os.makedirs(os.path.dirname(wsdl_exp_abs_path), exist_ok=True)
        with open(wsdl_exp_abs_path, "w", encoding="utf-8") as f:
            f.write(st.session_state["archivo_wsdl_exp"])


        # Guardar XSD respetando la ruta definida en ubicacion_xsd_exp
        xsd_rel_path = st.session_state["ubicacion_xsd_exp"]  # ej: "ComponentesComunes\\Resources\\Schemas\\Servicios\\DatosVisacionV2.1\\consultarInfoArchivoIngresoPrestamo.xsd"
        xsd_abs_path = os.path.join(tmpdir, xsd_rel_path)  # Se crea directamente en la raíz del .zip

        os.makedirs(os.path.dirname(xsd_abs_path), exist_ok=True)
        with open(xsd_abs_path, "w", encoding="utf-8") as f:
            f.write(st.session_state["xsd_file"])
        
        #Se guarda XSD ABC cuando es diferente operacion
        if st.session_state.get("misma_operacion_abc") == "NO":
            xsd_rel_path_abc = st.session_state["ubicacion_xsd_abc"]  # ej: "ComponentesComunes\\Resources\\Schemas\\Servicios\\DatosVisacionV2.1\\consultarInfoArchivoIngresoPrestamo.xsd"
            xsd_abs_path_abc = os.path.join(tmpdir, xsd_rel_path_abc)  # Se crea directamente en la raíz del .zip

            os.makedirs(os.path.dirname(xsd_abs_path_abc), exist_ok=True)
            with open(xsd_abs_path_abc, "w", encoding="utf-8") as f:
                f.write(st.session_state["xsd_file_abc"])
                
        # --- Crear estructura ABC ---
        abc_root = os.path.join(tmpdir, f"{st.session_state['nombre_capa_abc']}")
        
        proxy_abc_rel_path = st.session_state["ubicacion_proxy_abc"]
        proxy_abc_abs_path = os.path.join(tmpdir, proxy_abc_rel_path)  
        pipeline_abc_rel_path = st.session_state["ubicacion_pipeline_abc"]
        pipeline_abc_abs_path = os.path.join(tmpdir, pipeline_abc_rel_path)
        wsdl_abc_rel_path = st.session_state["ubicacion_wsdl_abc"]
        wsdl_abc_abs_path = os.path.join(tmpdir, wsdl_abc_rel_path) 

        #Proxy ABC
        os.makedirs(os.path.dirname(proxy_abc_abs_path), exist_ok=True)
        with open(proxy_abc_abs_path, "w", encoding="utf-8") as f:
            f.write(st.session_state["archivo_proxy_abc"])
            
        #Pipeline ABC
        os.makedirs(os.path.dirname(pipeline_abc_abs_path), exist_ok=True)
        with open(pipeline_abc_abs_path, "w", encoding="utf-8") as f:
            f.write(st.session_state["archivo_pipeline_abc"])
            
        #WSDL ABC
        os.makedirs(os.path.dirname(wsdl_abc_abs_path), exist_ok=True)
        with open(wsdl_abc_abs_path, "w", encoding="utf-8") as f:
            f.write(st.session_state["archivo_wsdl_abc"])
            
        if st.session_state.get("requiere_ebs") == "SI":
            # --- Crear estructura EBS ---
            ebs_root = os.path.join(tmpdir, f"{st.session_state['capa_seleccionada_ebs']}")
            
            proxy_ebs_rel_path = st.session_state["ubicacion_proxy_ebs"]
            proxy_ebs_abs_path = os.path.join(tmpdir, proxy_ebs_rel_path)  
            pipeline_ebs_rel_path = st.session_state["ubicacion_pipeline_ebs"]
            pipeline_ebs_abs_path = os.path.join(tmpdir, pipeline_ebs_rel_path)
            wsdl_ebs_rel_path = st.session_state["ubicacion_wsdl_ebs"]
            wsdl_ebs_abs_path = os.path.join(tmpdir, wsdl_ebs_rel_path) 

            #Proxy EBS
            os.makedirs(os.path.dirname(proxy_ebs_abs_path), exist_ok=True)
            with open(proxy_ebs_abs_path, "w", encoding="utf-8") as f:
                f.write(st.session_state["archivo_proxy_ebs"])
                
            #Pipeline EBS
            os.makedirs(os.path.dirname(pipeline_ebs_abs_path), exist_ok=True)
            with open(pipeline_ebs_abs_path, "w", encoding="utf-8") as f:
                f.write(st.session_state["archivo_pipeline_ebs"])
                
            #WSDL EBS
            os.makedirs(os.path.dirname(wsdl_ebs_abs_path), exist_ok=True)
            with open(wsdl_ebs_abs_path, "w", encoding="utf-8") as f:
                f.write(st.session_state["archivo_wsdl_ebs"])

        # --- Comprimir en .zip ---
        zip_base = os.path.join(tempfile.gettempdir(), st.session_state['service_name'])
        zip_path = shutil.make_archive(zip_base, 'zip', tmpdir)

        with open(zip_path, "rb") as fp:
            st.download_button(
                label="⬇️ Descargar Proyecto ZIP",
                data=fp,
                file_name=f"{st.session_state['service_name']}-{st.session_state["operation_name"]}.zip",
                mime="application/zip"
            )

# 🔧 Generador de bloques <imp:exportedItemInfo>
def generar_exported_item(instance_id, type_id, extrefs=None):
    info = RESOURCE_MAP[type_id]
    
    # Quitar extensión del instance_id si existe
    for ext in [".proxy", ".pipeline", ".Pipeline", ".WSDL", ".xsd"]:
        if instance_id.endswith(ext):
            instance_id = instance_id.replace(ext, "")
    
    # jarentryname = instanceId + .ExtensionOSB
    jarentryname = f"{instance_id}.{info['ext']}"
    
    # Armar referencias si existen
    extrefs_xml = ""
    if extrefs:
        extrefs_xml = "\n".join(
            [f'        <imp:property name="extrefs" value="{ref}"/>' for ref in extrefs]
        )
    
    return f"""
    <imp:exportedItemInfo instanceId="{instance_id}" typeId="{type_id}">
        <imp:properties>
            <imp:property name="representationversion" value="0"/>
            <imp:property name="dataclass" value="{info['dataclass']}"/>
            <imp:property name="isencrypted" value="false"/>
            <imp:property name="jarentryname" value="{jarentryname}"/>
{extrefs_xml}
        </imp:properties>
    </imp:exportedItemInfo>"""

def convertir_a_jarentryname(rel_path: str) -> str:
    """Convierte nombres como archivo.wsdl -> archivo.WSDL para el .jar de OSB"""
    dirname, filename = os.path.split(rel_path)
    for ext, tipo in type_extensions.items():
        if filename.endswith(ext):
            base = filename[: -len(ext)]
            new_filename = base + "." + tipo
            return os.path.join(dirname, new_filename).replace("\\", "/")
    return rel_path.replace("\\", "/")  # si no matchea, lo dejamos igualdef convertir_a_jarentryname(rel_path: str) -> str:
    """Convierte nombres como archivo.wsdl -> archivo.WSDL para el .jar de OSB"""
    dirname, filename = os.path.split(rel_path)
    for ext, tipo in type_extensions.items():
        if filename.endswith(ext):
            base = filename[: -len(ext)]
            new_filename = base + "." + tipo
            return os.path.join(dirname, new_filename).replace("\\", "/")
    return rel_path.replace("\\", "/")  # si no matchea, lo dejamos igual

def wrap_wsdl(file_path):
    with open(file_path, "r", encoding="utf-8") as f:
        content = f.read()
    wrapped = f'''<?xml version="1.0" encoding="UTF-8"?>
<con:wsdlEntry xmlns:con="http://www.bea.com/wli/sb/resources/config">
    <con:wsdl><![CDATA[{content}]]></con:wsdl>
    <con:dependencies/>
    <con:targetNamespace></con:targetNamespace>
</con:wsdlEntry>
'''
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(wrapped)


def wrap_xsd(file_path):
    with open(file_path, "r", encoding="utf-8") as f:
        content = f.read()
    wrapped = f'''<?xml version="1.0" encoding="UTF-8"?>
<con:schemaEntry xmlns:con="http://www.bea.com/wli/sb/resources/config">
    <con:schema><![CDATA[{content}]]></con:schema>
    <con:dependencies/>
    <con:targetNamespace></con:targetNamespace>
</con:schemaEntry>
'''
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(wrapped)

# 🚀 Exportar en formato .jar para JDeveloper
def exportar_proyecto_jar():
    with tempfile.TemporaryDirectory() as tmpdir:
        def guardar_recurso(rel_path, content):
            abs_path = os.path.join(tmpdir, rel_path)
            os.makedirs(os.path.dirname(abs_path), exist_ok=True)
            with open(abs_path, "w", encoding="utf-8") as f:
                f.write(content)

        # EXP
        if st.session_state["tipo_servicio"] == "Nuevo":
            guardar_recurso(st.session_state["ubicacion_proxy_exp"], st.session_state["archivo_proxy_exp"])
        guardar_recurso(st.session_state["ubicacion_pipeline_exp"], st.session_state["archivo_pipeline_exp"])
        guardar_recurso(st.session_state["ubicacion_wsdl_exp"], st.session_state["archivo_wsdl_exp"])
        guardar_recurso(st.session_state["ubicacion_xsd_exp"], st.session_state["xsd_file"])
        
        # ABC
        guardar_recurso(st.session_state["ubicacion_proxy_abc"], st.session_state["archivo_proxy_abc"])
        guardar_recurso(st.session_state["ubicacion_pipeline_abc"], st.session_state["archivo_pipeline_abc"])
        guardar_recurso(st.session_state["ubicacion_wsdl_abc"], st.session_state["archivo_wsdl_abc"])

        #Opcion operacion ABC diferente
        if st.session_state.get("misma_operacion_abc") == "NO":
            guardar_recurso(st.session_state["ubicacion_xsd_abc"], st.session_state["xsd_file_abc"])
        else:
            st.session_state["ubicacion_xsd_abc"] = st.session_state["ubicacion_xsd_exp"]
        # --- Construir ExportInfo dinámicamente ---
        exporttime = datetime.now().strftime("%a %b %d %H:%M:%S %Z %Y")
        items = []

        # --- PROXY ABC ---
        proxy_abc_path = st.session_state["ubicacion_proxy_abc"].replace("\\", "/").replace(".proxy", "")
        pipeline_abc_path = st.session_state["ubicacion_pipeline_abc"].replace("\\", "/").replace(".pipeline", "")
        wsdl_abc_path = st.session_state["ubicacion_wsdl_abc"].replace("\\", "/").replace(".wsdl", "")
        
        items.append(generar_exported_item(
            instance_id=proxy_abc_path,
            type_id="ProxyService",
            extrefs=[
                "Pipeline$" + pipeline_abc_path.replace("/", "$"),
                "WSDL$" + wsdl_abc_path.replace("/", "$")
            ]
        ))

        # --- PIPELINE ABC ---
        items.append(generar_exported_item(
            instance_id=pipeline_abc_path,
            type_id="Pipeline",
            extrefs=[
                "WSDL$" + wsdl_abc_path.replace("/", "$"),
                "ProxyService$ComponentesComunes$Proxies$PS_ManejadorGenericoErroresV1.0",
                "ProxyService$UtilitariosEBS$Proxies$AuditoriaSOA$RegistrarAuditoriaSOADATV1.0",
                "Xquery$ComponentesComunes$Resources$XQUERYs$xq_operacion_to_manejarError",
                "Xquery$ComponentesComunes$Resources$XQUERYs$xq_Auditoria_to_RegistrarAuditoriaSOA"
            ]
        ))

        # --- WSDL ABC ---
        xsd_exp_path = st.session_state["ubicacion_xsd_abc"].replace("\\", "/").replace(".xsd", "")
        items.append(generar_exported_item(
            instance_id=wsdl_abc_path,
            type_id="WSDL",
            extrefs=["XMLSchema$" + xsd_exp_path.replace("/", "$")]
        ))
        
        # EBS
        if st.session_state.get("requiere_ebs") == "SI":
            guardar_recurso(st.session_state["ubicacion_proxy_ebs"], st.session_state["archivo_proxy_ebs"])
            guardar_recurso(st.session_state["ubicacion_pipeline_ebs"], st.session_state["archivo_pipeline_ebs"])
            guardar_recurso(st.session_state["ubicacion_wsdl_ebs"], st.session_state["archivo_wsdl_ebs"])
            
            # --- PROXY EBS ---
            proxy_ebs_path = st.session_state["ubicacion_proxy_ebs"].replace("\\", "/").replace(".proxy", "")
            pipeline_ebs_path = st.session_state["ubicacion_pipeline_ebs"].replace("\\", "/").replace(".pipeline", "")
            wsdl_ebs_path = st.session_state["ubicacion_wsdl_ebs"].replace("\\", "/").replace(".wsdl", "")
            
            items.append(generar_exported_item(
                instance_id=proxy_ebs_path,
                type_id="ProxyService",
                extrefs=[
                    "Pipeline$" + pipeline_ebs_path.replace("/", "$"),
                    "WSDL$" + wsdl_ebs_path.replace("/", "$")
                ]
            ))

            # --- PIPELINE EBS ---
            items.append(generar_exported_item(
                instance_id=pipeline_ebs_path,
                type_id="Pipeline",
                extrefs=[
                    "WSDL$" + wsdl_ebs_path.replace("/", "$"),
                    "ProxyService$ComponentesComunes$Proxies$PS_ManejadorGenericoErroresV1.0",
                    "ProxyService$UtilitariosEBS$Proxies$AuditoriaSOA$RegistrarAuditoriaSOADATV1.0",
                    "Xquery$ComponentesComunes$Resources$XQUERYs$xq_operacion_to_manejarError",
                    "Xquery$ComponentesComunes$Resources$XQUERYs$xq_Auditoria_to_RegistrarAuditoriaSOA"
                ]
            ))

            # --- WSDL EBS ---
            xsd_exp_path = st.session_state["ubicacion_xsd_exp"].replace("\\", "/").replace(".xsd", "")
            items.append(generar_exported_item(
                instance_id=wsdl_ebs_path,
                type_id="WSDL",
                extrefs=["XMLSchema$" + xsd_exp_path.replace("/", "$")]
            ))
            #FIN EBS#

        # --- XSD EXP ---
        xsd_exp_full = st.session_state["ubicacion_xsd_exp"].replace("\\", "/").replace(".xsd", "")
        items.append(generar_exported_item(
            instance_id=xsd_exp_full,
            type_id="XMLSchema",
            extrefs=[
                "XMLSchema$ComponentesComunes$Resources$Schemas$Entidades$ComunesV2.1$Cabeceras"
            ]
        ))
        
        if st.session_state.get("misma_operacion_abc") == "NO":
            
            # --- XSD ABC ---
            xsd_abc_full = st.session_state["ubicacion_xsd_abc"].replace("\\", "/").replace(".xsd", "")
            items.append(generar_exported_item(
                instance_id=xsd_abc_full,
                type_id="XMLSchema",
                extrefs=[
                    "XMLSchema$ComponentesComunes$Resources$Schemas$Entidades$ComunesV2.1$Cabeceras"
                ]
            ))
            

        # --- PROXY EXP (si aplica) ---
        if st.session_state["tipo_servicio"] == "Nuevo":
            proxy_exp_path = st.session_state["ubicacion_proxy_exp"].replace("\\", "/").replace(".proxy", "")
            pipeline_exp_path = st.session_state["ubicacion_pipeline_exp"].replace("\\", "/").replace(".pipeline", "")
            wsdl_exp_path = st.session_state["ubicacion_wsdl_exp"].replace("\\", "/").replace(".wsdl", "")

            items.append(generar_exported_item(
                instance_id=proxy_exp_path,
                type_id="ProxyService",
                extrefs=[
                    "Pipeline$" + pipeline_exp_path.replace("/", "$"),
                    "WSDL$" + wsdl_exp_path.replace("/", "$")
                ]
            ))

        # --- PIPELINE EXP ---
        pipeline_exp_path = st.session_state["ubicacion_pipeline_exp"].replace("\\", "/").replace(".pipeline", "")
        wsdl_exp_path = st.session_state["ubicacion_wsdl_exp"].replace("\\", "/").replace(".wsdl", "")
        proxy_abc_ref = "ProxyService$" + proxy_abc_path.replace("/", "$")

        items.append(generar_exported_item(
            instance_id=pipeline_exp_path,
            type_id="Pipeline",
            extrefs=[
                "WSDL$" + wsdl_exp_path.replace("/", "$"),
                proxy_abc_ref,
                "ProxyService$ComponentesComunes$Proxies$PS_ManejadorGenericoErroresV1.0",
                "ProxyService$UtilitariosEBS$Proxies$AuditoriaSOA$RegistrarAuditoriaSOADATV1.0",
                "Xquery$ComponentesComunes$Resources$XQUERYs$xq_operacion_to_manejarError",
                "Xquery$ComponentesComunes$Resources$XQUERYs$xq_Auditoria_to_RegistrarAuditoriaSOA"
            ]
        ))

        # --- WSDL EXP ---
        items.append(generar_exported_item(
            instance_id=wsdl_exp_path,
            type_id="WSDL",
            extrefs=["XMLSchema$" + xsd_exp_path.replace("/", "$")]
        ))

        # --- Generar ExportInfo ---
        exported_items_xml = "\n".join(items)
        exportinfo_content = f"""<?xml version="1.0" encoding="UTF-8"?>
<xml-fragment name="OSB-IDE_build_{int(datetime.now().timestamp()*1000)}" version="v2" xmlns:imp="http://www.bea.com/wli/config/importexport">
    <imp:properties>
        <imp:property name="username" value="ServiceBus"/>
        <imp:property name="description" value=""/>
        <imp:property name="exporttime" value="{exporttime}"/>
        <imp:property name="productname" value="Oracle Service Bus"/>
        <imp:property name="productversion" value="12.2.1.3.0"/>
        <imp:property name="projectLevelExport" value="false"/>
    </imp:properties>
    {exported_items_xml}
</xml-fragment>
"""

        # Guardar ExportInfo
        exportinfo_path = os.path.join(tmpdir, "ExportInfo")
        with open(exportinfo_path, "w", encoding="utf-8") as f:
            f.write(exportinfo_content)

        # --- Comprimir como JAR ---
        jar_buffer = io.BytesIO()
        with zipfile.ZipFile(jar_buffer, "w", zipfile.ZIP_DEFLATED) as jar:
            for root, _, files in os.walk(tmpdir):
                for file in files:
                    abs_path = os.path.join(root, file)
                    rel_path = os.path.relpath(abs_path, tmpdir).replace("\\", "/")
                    
                    # 🔄 envolver si es WSDL o XSD
                    if file.lower().endswith(".wsdl"):
                        wrap_wsdl(abs_path)
                    elif file.lower().endswith(".xsd"):
                        wrap_xsd(abs_path)

                    # 🔄 convertir al formato OSB (extensión correcta)
                    jarentryname = convertir_a_jarentryname(rel_path)

                    # 📦 meter al jar
                    jar.write(abs_path, jarentryname)

        st.download_button(
            label="⬇️ Descargar Proyecto JAR",
            data=jar_buffer.getvalue(),
            file_name=f"{st.session_state['service_name']}-{st.session_state["operation_name"]}.jar",
            mime="application/java-archive"
        )
# -------------------------------
# App Streamlit
# -------------------------------

def generar_proyecto():
    
    # --- Inicializar valores solo si no existen ---
    if "ubicacion_xsd_exp" not in st.session_state:
        st.session_state["ubicacion_xsd_exp"] = ""
        
    if "nombre_capa_abc" not in st.session_state:
        st.session_state["nombre_capa_abc"] = ""
    
    if st.session_state["generar_proyecto"]:
        
        if "ubicacion_xsd_exp" in st.session_state:
            st.session_state["ubicacion_xsd_exp"] = ""
            
        if "ubicacion_xsd_abc" in st.session_state:
            st.session_state["ubicacion_xsd_abc"] = ""
        
        if (st.session_state["tipo_servicio"] == "Existente" and st.session_state["jar_file"] and st.session_state["operation_name"]) or (st.session_state["tipo_servicio"] == "Nuevo" and st.session_state["service_name"] and st.session_state["operation_name"] and (st.session_state["exp_proyecto"] and st.session_state["nombre_capa_abc"])):
            st.markdown(
            "<h3 style='text-align: center;'>📄 Generador de artefactos OSB según lineamientos</h3>",
            unsafe_allow_html=True)
            st.markdown(f"<p style='text-align: center;'>Servicio: {st.session_state["service_name"]} | Operación: {st.session_state["operation_name"]}</p>", unsafe_allow_html=True)
            
            if (st.session_state["tipo_servicio"] == "Existente"):
                st.session_state["btn_generar_capas"] = False
            
            if (st.session_state["tipo_servicio"] == "Nuevo"):
                st.session_state["btn_generar_capas"] = False
            
            
            if "targetnamespace" not in st.session_state:
                st.session_state["targetnamespace"] = f"http://xmlns.bancocajasocial.com/co/schemas/operacion/{st.session_state['operation_name']}/v1.0"
            
            if st.session_state.get("misma_operacion_abc") == "NO":
                
                if "targetnamespace_abc" not in st.session_state:
                    st.session_state["targetnamespace_abc"] = f"http://xmlns.bancocajasocial.com/co/schemas/operacion/{st.session_state['operation_name_abc']}/v1.0"
                
                if "xmlns_abc" not in st.session_state:
                    st.session_state["xmlns_abc"] = generar_xmlns(st.session_state["operation_name_abc"])
                    
                st.session_state["complextype_abc"] = capitalizar_inicio(st.session_state["operation_name_abc"])
                st.session_state["xsd_name_abc"] = st.session_state["operation_name_abc"] +".xsd"
                
                st.session_state["xsd_file_abc"] = generate_xsd(st.session_state["operation_name_abc"],st.session_state["complextype_abc"],st.session_state["xmlns_abc"])
                st.session_state["wsdl_text_abc"] = ""
                
                st.session_state["input_xsd_abc"]  = st.session_state["operation_name_abc"]+"Request"
                st.session_state["output_xsd_abc"] = st.session_state["operation_name_abc"]+"Response"
                
            
            #targetnamespace = f"http://xmlns.bancocajasocial.com/co/schemas/operacion/{st.session_state["operation_name"]}"
            if "xmlns" not in st.session_state:
                st.session_state["xmlns"] = generar_xmlns(st.session_state["operation_name"])
            
            
            xmlns = generar_xmlns(st.session_state["operation_name"])
            
            # col1, col2 = st.columns(2)
            # with col1:
            # --- Inputs persistentes ---

            complextype = capitalizar_inicio(st.session_state["operation_name"])
            st.session_state["xsd_name"] = st.session_state["operation_name"] +".xsd"
            
            st.session_state["xsd_file"] = generate_xsd(st.session_state["operation_name"],complextype,xmlns)
            wsdl_text = ""
            
            input_xsd = st.session_state["operation_name"]+"Request"
            output_xsd = st.session_state["operation_name"]+"Response"
            
            st.session_state["input_xsd"] = input_xsd
            st.session_state["output_xsd"] = output_xsd
            
            #XSD EXP
            with st.expander("⚙️ Configuración XSD EXP", expanded=True):

                #st.markdown("<h6 style='text-align: center;'>Nombre XSD</h6>", unsafe_allow_html=True)
                st.markdown(f"<h6 style='text-align: center;'>{st.session_state["xsd_name"]}</h6>", unsafe_allow_html=True)
                st.text_input("📝 targetNamespace", disabled=True, key="targetnamespace")
                st.text_input("📝 xmlns", value=st.session_state["xmlns"], disabled=True)
                
                # # 👇 caja con pestañas
                # tab1, tab2 = st.tabs(["📄 XSD File", "📌 Otra info"])
                # with tab1:
                    # st.code(st.session_state["xsd_file"], language="xml")
                # with tab2:
                    # st.write("Aquí puedes poner más cosas relacionadas")
                #st.code(st.session_state["xsd_file"].replace("\n",""), language="xml")

                # Campo editable que recuerda su valor
                st.session_state["ubicacion_xsd_exp"] = st.text_input(
                    "📝 Ubicación Raíz XSD EXP (Ejemplo: 'ComponentesComunes\Resources\Schemas\Servicios\BPMV2.1'",
                    value=st.session_state["ubicacion_xsd_exp"],  # recupera siempre
                    key="ubicacion_xsd_exp_input"
                )
                
                
                
                if not st.session_state["ubicacion_xsd_exp"]:
                    st.warning("⚠ Digita la ubicacion del xsd.")
            
            #XSD ABC (Opcional)
            if st.session_state.get("misma_operacion_abc") == "NO":
                
                
                with st.expander("⚙️ Configuración XSD ABC", expanded=True):

                    #st.markdown("<h6 style='text-align: center;'>Nombre XSD</h6>", unsafe_allow_html=True)
                    st.markdown(f"<h6 style='text-align: center;'>{st.session_state["xsd_name_abc"]}</h6>", unsafe_allow_html=True)
                    st.text_input("📝 targetNamespace", disabled=True, key="targetnamespace_abc")
                    st.text_input("📝 xmlns", value=st.session_state["xmlns_abc"], disabled=True)
                    
                    ubicacion_xsd_abc = st.session_state.get("ubicacion_xsd_abc", "")
                    
                    # Campo editable que recuerda su valor
                    st.session_state["ubicacion_xsd_abc"] = st.text_input(
                        "📝 Ubicación XSD ABC",
                        value=ubicacion_xsd_abc,  # recupera siempre
                        key="ubicacion_xsd_abc_input"
                    )

                    if not st.session_state["ubicacion_xsd_abc"]:
                        st.warning("⚠ Digita la ubicacion del xsd ABC.")
            
            
            if st.session_state["tipo_servicio"] == "Existente" and st.session_state["ubicacion_xsd_exp"]:
                
                if st.session_state.get("misma_operacion_abc") == "NO" and st.session_state["ubicacion_xsd_abc"] or st.session_state.get("misma_operacion_abc") == "SI":
                
                    if "wsdl" not in st.session_state:
                        st.session_state["wsdl"] = ""
                    else:
                        wsdl_exp = st.session_state["wsdl"] +".wsdl"
                        
                    if "ruta_wsdl" not in st.session_state:
                        st.session_state["ruta_wsdl"] = ""

                    if st.session_state["ruta_wsdl"]:
                        wsdl_completa = st.session_state["ruta_wsdl"] +".WSDL"
                        try:
                            wsdl_text = leer_wsdl(st.session_state["jar_file"], wsdl_completa)
                            wsdl_text = limpiar_wsdl_contenido(wsdl_text)
                            st.session_state["wsdl_text"] = wsdl_text
                            st.session_state["wsdl_completa"] = wsdl_completa
                        except Exception as e:
                            st.error(f"❌ Error al procesar el WSDL {st.session_state['ruta_wsdl']}: {e}")
                    
                    else:
                        st.warning("⚠ No se encontró ruta válida para el WSDL.")
                    
                    
                    if st.session_state["ruta_pipeline_exp"]:
                        pipeline_completa = st.session_state["ruta_pipeline_exp"] +".Pipeline"
                        try:
                            pipeline_text = leer_pipeline(st.session_state["jar_file"], pipeline_completa)
                            #pipeline_text = limpiar_pipeline_contenido(pipeline_text)
                            st.session_state["pipeline_text"] = pipeline_text
                            st.session_state["pipeline_completa"] = pipeline_completa
                        except Exception as e:
                            st.error(f"❌ Error al procesar el Pipeline {st.session_state['ruta_pipeline_exp']}: {e}")
                    
                    else:
                        st.warning("⚠ No se encontró ruta válida para el Pipeline.")
                    
                    if "btn_actualizar_servicio" not in st.session_state:
                        st.session_state["btn_actualizar_servicio"] = False

                    if st.button("Actualizar Sevicio"):
                        st.session_state["btn_actualizar_servicio"] = True

                    if st.session_state["btn_actualizar_servicio"]:

                        st.session_state["ubicacion_xsd_exp"] = f"{st.session_state['ubicacion_xsd_exp']}\\{st.session_state['xsd_name']}"
                        
                        if st.session_state.get("misma_operacion_abc") == "NO":
                            st.session_state["ubicacion_xsd_abc"] = f"{st.session_state['ubicacion_xsd_abc']}\\{st.session_state['xsd_name_abc']}"
                        else:
                            st.session_state["ubicacion_xsd_abc"] = st.session_state["ubicacion_xsd_exp"]
                            st.session_state["targetnamespace_abc"] = st.session_state["targetnamespace"]
                            st.session_state["xmlns_abc"] = st.session_state["xmlns"]
                            
                        
                        with st.expander("⚙️Generacion capa ABC", expanded=True):
                        
                            st.markdown(f"<h6 style='text-align: center;'>{generar_nombrado_abc(st.session_state["operation_name_abc"], "nombre", st.session_state["version_proxy_abc"])}</h6>", unsafe_allow_html=True)
                            
                            st.session_state["proxy_abc"] = generar_nombrado_abc(st.session_state["operation_name_abc"], "proxy", st.session_state["version_proxy_abc"])
                            st.session_state["ubicacion_proxy_abc"] = st.session_state["nombre_capa_abc"]+"/Proxies/"+st.session_state["proxy_abc"]
                            st.session_state["pipeline_abc"] = generar_nombrado_abc(st.session_state["operation_name_abc"], "pipeline", st.session_state["version_proxy_abc"])
                            st.session_state["ubicacion_pipeline_abc"] = st.session_state["nombre_capa_abc"]+"/Pipeline/"+st.session_state["pipeline_abc"]
                            st.session_state["wsdl_abc"] = generar_nombrado_abc(st.session_state["operation_name_abc"], "wsdl", st.session_state["version_proxy_abc"])
                            st.session_state["ubicacion_wsdl_abc"] = st.session_state["nombre_capa_abc"]+"/Resources/WSDLs/"+st.session_state["wsdl_abc"]
                            
                            
                            st.session_state["archivo_wsdl_abc"] = crear_wsdl_abc(
                                    st.session_state["operation_name_abc"],
                                    st.session_state["ubicacion_wsdl_abc"],
                                    st.session_state["ubicacion_xsd_abc"],
                                    st.session_state["targetnamespace_abc"],
                                    st.session_state["xmlns_abc"]
                                )
                            
                            st.session_state["namespace_wsdl_abc"], st.session_state["binding_wsdl_abc"] = obtener_namespace_y_binding(st.session_state["archivo_wsdl_abc"])
                            
                            st.markdown(
                                    f"""
                                    <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Proxy ABC</div>
                                    <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_proxy_abc"]}</div>
                                    """,
                                    unsafe_allow_html=True
                            )
                            st.text_input("📝 Proxy ABC", value=st.session_state["proxy_abc"], disabled=True, label_visibility="collapsed")
                            
                            st.session_state["archivo_proxy_abc"] = crear_proxy_abc(
                                quitar_extension(st.session_state["ubicacion_wsdl_abc"]),
                                st.session_state["binding_wsdl_abc"],
                                st.session_state["namespace_wsdl_abc"],
                                quitar_extension(st.session_state["ubicacion_pipeline_abc"])
                            )
                            
                            #st.code(st.session_state["archivo_proxy_abc"].replace("\n", " "), language="xml")
                            
                            
                            st.markdown(
                                    f"""
                                    <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Pipeline ABC</div>
                                    <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_pipeline_abc"]}</div>
                                    """,
                                    unsafe_allow_html=True
                            )
                            st.text_input("📝 Pipeline ABC", value=st.session_state["pipeline_abc"], disabled=True, label_visibility="collapsed")
                            
                            
                            st.session_state["archivo_pipeline_abc"] = crear_pipeline_abc(
                                quitar_extension(st.session_state["ubicacion_wsdl_abc"]),
                                st.session_state["binding_wsdl_abc"],
                                st.session_state["namespace_wsdl_abc"],
                                st.session_state["operation_name_abc"]
                            )
                            
                            #st.code(st.session_state["archivo_pipeline_abc"].replace("\n", " "), language="xml")
                            
                            st.markdown(
                                    f"""
                                    <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 WSDL ABC</div>
                                    <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_wsdl_abc"]}</div>
                                    """,
                                    unsafe_allow_html=True
                            )
                            st.text_input("📝 WSDL ABC", value=st.session_state["wsdl_abc"], disabled=True, label_visibility="collapsed")
                            
                        if st.session_state.get("requiere_ebs") == "SI":
                            with st.expander("⚙️Generacion capa EBS", expanded=True):
                            
                                st.markdown(f"<h6 style='text-align: center;'>{generar_nombrado_ebs(st.session_state["operation_name"], "nombre", st.session_state["version_ebs"])}</h6>", unsafe_allow_html=True)
                                
                                #st.write(f"{st.session_state["capa_seleccionada_ebs"]}")
                                st.session_state["proxy_ebs"] = generar_nombrado_ebs(st.session_state["operation_name"], "proxy", st.session_state["version_ebs"])
                                st.session_state["ubicacion_proxy_ebs"] = st.session_state["capa_seleccionada_ebs"].split('/')[0]+"/Proxies/"+st.session_state["proxy_ebs"]
                                st.session_state["pipeline_ebs"] = generar_nombrado_ebs(st.session_state["operation_name"], "pipeline", st.session_state["version_ebs"])
                                st.session_state["ubicacion_pipeline_ebs"] = st.session_state["capa_seleccionada_ebs"].split('/')[0]+"/Pipeline/"+st.session_state["pipeline_ebs"]
                                st.session_state["wsdl_ebs"] = generar_nombrado_ebs(st.session_state["operation_name"], "wsdl", st.session_state["version_ebs"])
                                st.session_state["ubicacion_wsdl_ebs"] = st.session_state["capa_seleccionada_ebs"].split('/')[0]+"/Resources/Wsdls/"+st.session_state["wsdl_ebs"]
                                
                                
                                st.session_state["archivo_wsdl_ebs"] = crear_wsdl_ebs(
                                        st.session_state["operation_name"],
                                        st.session_state["ubicacion_wsdl_ebs"],
                                        st.session_state["ubicacion_xsd_exp"],
                                        st.session_state["targetnamespace"],
                                        st.session_state["xmlns"]
                                    )
                                
                                st.session_state["namespace_wsdl_ebs"], st.session_state["binding_wsdl_ebs"] = obtener_namespace_y_binding(st.session_state["archivo_wsdl_ebs"])
                                
                                st.markdown(
                                        f"""
                                        <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Proxy EBS</div>
                                        <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_proxy_ebs"]}</div>
                                        """,
                                        unsafe_allow_html=True
                                )
                                st.text_input("📝 Proxy EBS", value=st.session_state["proxy_ebs"], disabled=True, label_visibility="collapsed")
                                
                                st.session_state["archivo_proxy_ebs"] = crear_proxy_ebs(
                                    quitar_extension(st.session_state["ubicacion_wsdl_ebs"]),
                                    st.session_state["binding_wsdl_ebs"],
                                    st.session_state["namespace_wsdl_ebs"],
                                    quitar_extension(st.session_state["ubicacion_pipeline_ebs"])
                                )
                                
                                #st.code(st.session_state["archivo_proxy_ebs"].replace("\n", " "), language="xml")
                                
                                
                                st.markdown(
                                        f"""
                                        <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Pipeline EBS</div>
                                        <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_pipeline_ebs"]}</div>
                                        """,
                                        unsafe_allow_html=True
                                )
                                st.text_input("📝 Pipeline EBS", value=st.session_state["pipeline_ebs"], disabled=True, label_visibility="collapsed")
                                
                                
                                st.session_state["archivo_pipeline_ebs"] = crear_pipeline_ebs(
                                    quitar_extension(st.session_state["ubicacion_wsdl_ebs"]),
                                    st.session_state["binding_wsdl_ebs"],
                                    st.session_state["namespace_wsdl_ebs"],
                                    st.session_state["operation_name"]
                                )
                                
                                #st.code(st.session_state["archivo_pipeline_ebs"].replace("\n", " "), language="xml")
                                
                                st.markdown(
                                        f"""
                                        <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 WSDL EBS</div>
                                        <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_wsdl_ebs"]}</div>
                                        """,
                                        unsafe_allow_html=True
                                )
                                st.text_input("📝 WSDL EBS", value=st.session_state["wsdl_ebs"], disabled=True, label_visibility="collapsed")
                                

                        with st.expander("⚙️Actualizacion capa EXP", expanded=True):
                            
                            st.markdown(f"<h6 style='text-align: center;'>{st.session_state["service_name"]}</h6>", unsafe_allow_html=True)
                            
                            st.session_state["proxy_exp"] = generar_nombrado_exp(st.session_state["service_name"], "proxy")
                            st.session_state["ubicacion_proxy_exp"] = st.session_state["nombre_capa_exp"]+"/Proxies/"+st.session_state["proxy_exp"]
                            st.session_state["pipeline_exp"] = generar_nombrado_exp(st.session_state["service_name"], "pipeline")
                            st.session_state["ubicacion_pipeline_exp"] = st.session_state["nombre_capa_exp"]+"/Pipeline/"+st.session_state["pipeline_exp"]
                            st.session_state["wsdl_exp"] = generar_nombrado_exp(st.session_state["service_name"], "wsdl")
                            st.session_state["ubicacion_wsdl_exp"] = st.session_state["nombre_capa_exp"]+"/Resources/Wsdls/"+st.session_state["wsdl_exp"]
                            
                            st.session_state["archivo_wsdl_exp"] = procesar_wsdl(
                                st.session_state["wsdl_text"],
                                st.session_state["wsdl_completa"],
                                st.session_state["targetnamespace"],
                                st.session_state["ubicacion_xsd_exp"],
                                st.session_state["operation_name"],
                                st.session_state["input_xsd"],
                                st.session_state["output_xsd"],
                                st.session_state["xmlns"]
                            )
                            
                            #st.code(st.session_state["wsdl_text"], language="xml")
                            #st.code(st.session_state["archivo_wsdl_exp"], language="xml")
                            xml_debug = st.session_state["archivo_wsdl_exp"].strip()
                            # Quitar BOM si existe
                            if xml_debug.startswith("\ufeff"):
                                xml_debug = xml_debug.encode('utf-8').decode('utf-8-sig')

                            try:
                                ET.fromstring(xml_debug)
                                #st.success("XML parsea correctamente")
                            except ET.ParseError as e:
                                error_message = str(e)
                                # Validar si el error es por atributo duplicado
                                if "duplicate attribute" in error_message.lower():
                                    st.error("🚫 Error: Ya existe una operación con el mismo nombre en el WSDL actual. "
                                             "Por favor, cambia el nombre de la operación e inténtalo nuevamente.")
                                    st.session_state["archivo_wsdl_exp"] = st.session_state["wsdl_text"]
                                else:
                                    st.error(f"XML inválido: {error_message}")
                                
                                # Mostrar el XML problemático siempre que haya error
                                with st.expander("Ver XML problemático", expanded=True):
                                    st.code(xml_debug, language="xml")

                            st.session_state["namespace_wsdl_exp"], st.session_state["binding_wsdl_exp"] = obtener_namespace_y_binding(st.session_state["archivo_wsdl_exp"])
                            
                            #st.code(st.session_state["namespace_wsdl_exp"], language="xml")
                            #st.code(st.session_state["binding_wsdl_exp"], language="xml")
                            

                            st.markdown(
                                f"""
                                <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Proxy EXP</div>
                                <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_proxy_exp"]}</div>
                                """,
                                unsafe_allow_html=True
                            )
                            
                            st.text_input("📝 Proxy EXP", value=st.session_state["proxy_exp"], disabled=True, label_visibility="collapsed")
                            
                            st.session_state["archivo_proxy_exp"] = crear_proxy_exp(
                                st.session_state["proxy_exp"],
                                quitar_extension(st.session_state["ubicacion_wsdl_exp"]),
                                st.session_state["binding_wsdl_exp"],
                                st.session_state["namespace_wsdl_exp"],
                                quitar_extension(st.session_state["ubicacion_pipeline_exp"]),
                                st.session_state["service_name"]
                            )

                            #st.code(st.session_state["archivo_proxy_exp"].replace("\n", " "), language="xml")
                            
                            st.markdown(
                                f"""
                                <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Pipeline EXP</div>
                                <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_pipeline_exp"]}</div>
                                """,
                                unsafe_allow_html=True
                            )
                    
                            st.text_input("📝 Pipeline EXP", value=st.session_state["pipeline_exp"], disabled=True, label_visibility="collapsed")
                            
                            
                            st.session_state["archivo_pipeline_exp"] = agregar_operacion_pipeline(
                                st.session_state["pipeline_text"],
                                st.session_state["operation_name"],
                                st.session_state["targetnamespace"],
                                os.path.normpath(st.session_state["ubicacion_xsd_exp"]).rsplit('.', 1)[0].replace("\\", "/"),
                                os.path.normpath(st.session_state["ubicacion_proxy_abc"]).rsplit('.', 1)[0].replace("\\", "/")
                            )
                            #st.code(st.session_state["archivo_pipeline_exp"], language="xml")
                            
                            
                            st.markdown(
                                f"""
                                <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 WSDL EXP</div>
                                <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_wsdl_exp"]}</div>
                                """,
                                unsafe_allow_html=True
                            )

                            st.text_input("📝 WSDL EXP", value=st.session_state["wsdl_exp"], disabled=True, label_visibility="collapsed")

                            
                            st.session_state["wsdl_completa"] = st.session_state["ubicacion_wsdl_exp"] + st.session_state["wsdl_exp"]

                            #st.markdown("<h6 style='text-align: left;'>📝WSDL Autogenerado:</h6>", unsafe_allow_html=True)
                            #st.code(st.session_state["archivo_wsdl_exp"], language="xml")

            # Botón que solo cambia el estado
            if st.session_state["tipo_servicio"] == "Nuevo" and st.session_state["ubicacion_xsd_exp"]:
                #col1, col2, col3 = st.columns([1,2,1])  # la del medio es más grande
                #with col2:
                
                if "btn_generar_capas" not in st.session_state:
                    st.session_state["btn_generar_capas"] = False

                if st.button("Generar capas"):
                    st.session_state["btn_generar_capas"] = True

                if st.session_state["btn_generar_capas"]:
                    
                    st.session_state["ubicacion_xsd_exp"] = f"{st.session_state['ubicacion_xsd_exp']}\\{st.session_state['xsd_name']}"

                    with st.expander("⚙️Generacion capa ABC", expanded=True):
                    
                        st.markdown(f"<h6 style='text-align: center;'>{generar_nombrado_abc(st.session_state["operation_name_abc"], "nombre", st.session_state["version_proxy_abc"])}</h6>", unsafe_allow_html=True)
                        
                        st.session_state["proxy_abc"] = generar_nombrado_abc(st.session_state["operation_name_abc"], "proxy", st.session_state["version_proxy_abc"])
                        st.session_state["ubicacion_proxy_abc"] = st.session_state["nombre_capa_abc"]+"/Proxies/"+st.session_state["proxy_abc"]
                        st.session_state["pipeline_abc"] = generar_nombrado_abc(st.session_state["operation_name_abc"], "pipeline", st.session_state["version_proxy_abc"])
                        st.session_state["ubicacion_pipeline_abc"] = st.session_state["nombre_capa_abc"]+"/Pipeline/"+st.session_state["pipeline_abc"]
                        st.session_state["wsdl_abc"] = generar_nombrado_abc(st.session_state["operation_name_abc"], "wsdl", st.session_state["version_proxy_abc"])
                        st.session_state["ubicacion_wsdl_abc"] = st.session_state["nombre_capa_abc"]+"/Resources/WSDLs/"+st.session_state["wsdl_abc"]
                        
                        
                        st.session_state["archivo_wsdl_abc"] = crear_wsdl_abc(
                                st.session_state["operation_name_abc"],
                                st.session_state["ubicacion_wsdl_abc"],
                                st.session_state["ubicacion_xsd_exp"],
                                st.session_state["targetnamespace"],
                                st.session_state["xmlns"]
                            )
                        
                        st.session_state["namespace_wsdl_abc"], st.session_state["binding_wsdl_abc"] = obtener_namespace_y_binding(st.session_state["archivo_wsdl_abc"])
                        
                        st.markdown(
                                f"""
                                <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Proxy ABC</div>
                                <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_proxy_abc"]}</div>
                                """,
                                unsafe_allow_html=True
                        )
                        st.text_input("📝 Proxy ABC", value=st.session_state["proxy_abc"], disabled=True, label_visibility="collapsed")
                        
                        st.session_state["archivo_proxy_abc"] = crear_proxy_abc(
                            quitar_extension(st.session_state["ubicacion_wsdl_abc"]),
                            st.session_state["binding_wsdl_abc"],
                            st.session_state["namespace_wsdl_abc"],
                            quitar_extension(st.session_state["ubicacion_pipeline_abc"])
                        )
                        
                        #st.code(st.session_state["archivo_proxy_abc"].replace("\n", " "), language="xml")
                        
                        
                        st.markdown(
                                f"""
                                <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Pipeline ABC</div>
                                <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_pipeline_abc"]}</div>
                                """,
                                unsafe_allow_html=True
                        )
                        st.text_input("📝 Pipeline ABC", value=st.session_state["pipeline_abc"], disabled=True, label_visibility="collapsed")
                        
                        
                        st.session_state["archivo_pipeline_abc"] = crear_pipeline_abc(
                            quitar_extension(st.session_state["ubicacion_wsdl_abc"]),
                            st.session_state["binding_wsdl_abc"],
                            st.session_state["namespace_wsdl_abc"],
                            st.session_state["operation_name_abc"]
                        )
                        
                        #st.code(st.session_state["archivo_pipeline_abc"].replace("\n", " "), language="xml")
                        
                        st.markdown(
                                f"""
                                <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 WSDL ABC</div>
                                <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_wsdl_abc"]}</div>
                                """,
                                unsafe_allow_html=True
                        )
                        st.text_input("📝 WSDL ABC", value=st.session_state["wsdl_abc"], disabled=True, label_visibility="collapsed")
                        
                        
                        #st.markdown("<h6 style='text-align: left;'>📝WSDL Autogenerado:</h6>", unsafe_allow_html=True)
                        #st.code(st.session_state["archivo_wsdl_abc"].replace("\n", " "), language="xml")
                    

                    with st.expander("⚙️Generacion capa EXP", expanded=True):
                        
                        st.markdown(f"<h6 style='text-align: center;'>{st.session_state["service_name"]}</h6>", unsafe_allow_html=True)
                        
                        st.session_state["proxy_exp"] = generar_nombrado_exp(st.session_state["service_name"], "proxy")
                        st.session_state["ubicacion_proxy_exp"] = st.session_state["exp_proyecto"]+"/Proxies/"+st.session_state["proxy_exp"]
                        st.session_state["pipeline_exp"] = generar_nombrado_exp(st.session_state["service_name"], "pipeline")
                        st.session_state["ubicacion_pipeline_exp"] = st.session_state["exp_proyecto"]+"/Pipeline/"+st.session_state["pipeline_exp"]
                        st.session_state["wsdl_exp"] = generar_nombrado_exp(st.session_state["service_name"], "wsdl")
                        st.session_state["ubicacion_wsdl_exp"] = st.session_state["exp_proyecto"]+"/Resources/Wsdls/"+st.session_state["wsdl_exp"]
                        
                        st.session_state["archivo_wsdl_exp"] = crear_wsdl_exp(
                            st.session_state["service_name"],
                            st.session_state["ubicacion_wsdl_exp"],
                            st.session_state["ubicacion_xsd_exp"],
                            st.session_state["operation_name"],
                            st.session_state["input_xsd"],
                            st.session_state["output_xsd"],
                            st.session_state["xmlns"],
                            st.session_state["targetnamespace"]
                        )
                        
                        st.session_state["namespace_wsdl_exp"], st.session_state["binding_wsdl_exp"] = obtener_namespace_y_binding(st.session_state["archivo_wsdl_exp"])
                        
                        #st.code(st.session_state["namespace_wsdl_exp"], language="xml")
                        #st.code(st.session_state["binding_wsdl_exp"], language="xml")
                        

                        st.markdown(
                            f"""
                            <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Proxy EXP</div>
                            <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_proxy_exp"]}</div>
                            """,
                            unsafe_allow_html=True
                        )
                        
                        st.text_input("📝 Proxy EXP", value=st.session_state["proxy_exp"], disabled=True, label_visibility="collapsed")
                        
                        st.session_state["archivo_proxy_exp"] = crear_proxy_exp(
                            st.session_state["proxy_exp"],
                            quitar_extension(st.session_state["ubicacion_wsdl_exp"]),
                            st.session_state["binding_wsdl_exp"],
                            st.session_state["namespace_wsdl_exp"],
                            quitar_extension(st.session_state["ubicacion_pipeline_exp"]),
                            st.session_state["service_name"]
                        )

                        #st.code(st.session_state["archivo_proxy_exp"].replace("\n", " "), language="xml")
                        
                        st.markdown(
                            f"""
                            <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 Pipeline EXP</div>
                            <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_pipeline_exp"]}</div>
                            """,
                            unsafe_allow_html=True
                        )
                
                        st.text_input("📝 Pipeline EXP", value=st.session_state["pipeline_exp"], disabled=True, label_visibility="collapsed")
                        
                        
                        st.session_state["archivo_pipeline_exp"] = crear_pipeline_exp(
                            st.session_state["pipeline_exp"],
                            quitar_extension(st.session_state["ubicacion_wsdl_exp"]),
                            st.session_state["binding_wsdl_exp"],
                            st.session_state["namespace_wsdl_exp"],
                            st.session_state["operation_name"],
                            st.session_state["ubicacion_proxy_abc"])
                        
                        #st.code(st.session_state["archivo_pipeline_exp"].replace("\n", " "), language="xml")
                        
                        
                        st.markdown(
                            f"""
                            <div style="font-size:14px; font-weight:400; font-family:Source Sans Pro">📝 WSDL EXP</div>
                            <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_wsdl_exp"]}</div>
                            """,
                            unsafe_allow_html=True
                        )

                        st.text_input("📝 WSDL EXP", value=st.session_state["wsdl_exp"], disabled=True, label_visibility="collapsed")

                        
                        st.session_state["wsdl_completa"] = st.session_state["ubicacion_wsdl_exp"] + st.session_state["wsdl_exp"]

                        #st.markdown("<h6 style='text-align: left;'>📝WSDL Autogenerado:</h6>", unsafe_allow_html=True)
                        #st.code(st.session_state["archivo_wsdl_exp"].replace("\n", " "), language="xml")
                    
            if (st.session_state["btn_actualizar_servicio"] or st.session_state["btn_generar_capas"]) and st.session_state["ubicacion_xsd_exp"]!="":
                st.success("✅ Archivos generados correctamente.")
                col1, col2, col3, col4 = st.columns([1, 1, 1, 1])

                with col2:
                    exportar_proyecto_zip()

                with col3:
                    exportar_proyecto_jar()

            #if st.session_state["btn_generar_capas"] or st.session_state["btn_actualizar_servicio"]:

def main():

    # Ruta donde se extraerán los archivos
    carpeta_destino = "extraccion_jar"
    
    if "ruta_wsdl" not in st.session_state:
        st.session_state["ruta_wsdl"] = None
    
    # Inicializar claves de session_state si no existen
    if "btn_generar_capas" not in st.session_state:
        st.session_state["btn_generar_capas"] = False

    if "btn_actualizar_servicio" not in st.session_state:
        st.session_state["btn_actualizar_servicio"] = False
        
    if "btn_generar_capa_abc" not in st.session_state:
        st.session_state["btn_generar_capa_abc"] = False
        
    # Inicializar claves de session_state si no existen
    if "operation_name" not in st.session_state:
        st.session_state["operation_name"] = ""
    
    if "xsd_name" not in st.session_state:
        st.session_state["xsd_name"] = ""
        
    if "proxy_abc" not in st.session_state:
        st.session_state["proxy_abc"] = ""
        
    if "pipeline_abc" not in st.session_state:
        st.session_state["pipeline_abc"] = ""
        
    if "wsdl_abc" not in st.session_state:
        st.session_state["wsdl_abc"] = ""
    
    if "version_proxy" not in st.session_state:
        st.session_state["version_proxy"] = ""
        
    if "version_proxy_abc" not in st.session_state:
        st.session_state["version_proxy_abc"] = ""
    
    if "nombre_capa_abc" not in st.session_state:
        st.session_state["nombre_capa_abc"] = ""
    
    if "nombre_capa_exp" not in st.session_state:
        st.session_state["nombre_capa_exp"] = ""
            
    if "ruta_pipeline_exp" not in st.session_state:
        st.session_state["ruta_pipeline_exp"] = ""
        
    # 📌 Agregar elementos al menú lateral
    with st.sidebar:
        st.markdown(f"<h3 style='text-align: center;'>Parámetros básicos OSB</h3>", unsafe_allow_html=True)
        
        # Radio para nuevo o existente
        st.session_state["tipo_servicio"] = st.radio(
            "¿El servicio a exponer es nuevo o existente?",
            ("Nuevo", "Existente")
        )

        if st.session_state["tipo_servicio"] == "Existente":
            jar_file = st.file_uploader("📦 Sube el archivo .jar del servicio existente (.proxy con dependencias)", type=["jar"])
            
            # Diccionario de capas con tipos de archivo dentro
            capas = ["EXP", "EBS", "ABC"]
            artefactos = ["Pipeline", "Proxy", "WSDL", "BusinessService"]

            # Inicializar estructura
            capas_detectadas = {capa: {tipo: [] for tipo in artefactos} for capa in capas}
            
            terminacion_ebs = ["DS", "AS"]
            
            st.session_state["jar_file"] = jar_file
            
            if jar_file:
                with zipfile.ZipFile(jar_file, "r") as jar:
                    rutas = jar.namelist()

                    for ruta in rutas:
                        ruta_norm = ruta.replace("\\", "/")
                        ruta_low = ruta_norm.lower()

                        if ruta_norm.endswith("/"):
                            continue  # saltar carpetas

                        # Detectar capa según segmentos de la ruta
                        capa_actual = None
                        for capa in capas:
                            if any(capa.lower() in segment for segment in ruta_low.split("/")):
                                capa_actual = capa
                                break

                        if not capa_actual:
                            continue  # si no pertenece a ninguna capa, saltar

                        # Detectar tipo de artefacto
                        if ruta_low.endswith(".pipeline"):
                            capas_detectadas[capa_actual]["Pipeline"].append(ruta)
                        elif ruta_low.endswith(".proxy") or ruta_low.endswith(".proxyservice") or "proxyservice" in ruta_low:
                            capas_detectadas[capa_actual]["Proxy"].append(ruta)
                        elif ruta_low.endswith(".wsdl"):
                            capas_detectadas[capa_actual]["WSDL"].append(ruta)
                        elif ruta_low.endswith(".businessservice"):
                            capas_detectadas[capa_actual]["BusinessService"].append(ruta)
                            
                proxies_exp = capas_detectadas["EXP"]["Proxy"]
                proxies_ebs = capas_detectadas["EBS"]["Proxy"]
                proxies_abc = capas_detectadas["ABC"]["Proxy"]
                
                # Carpeta raíz detectada
                carpetas_raiz = set()

                for ruta in rutas:
                    # Normalizar
                    ruta_norm = ruta.replace("\\", "/")
                    
                    # Tomar solo el primer segmento
                    carpeta = ruta_norm.split("/")[0]
                    
                    # Evitar agregar archivos sueltos o vacíos
                    if carpeta.strip():
                        carpetas_raiz.add(carpeta)
                
                carpetas_raiz = sorted(list(carpetas_raiz))

                excluir = {"ExportInfo", "UtilitariosEBS"}
                carpetas_raiz = [c for c in carpetas_raiz if c not in excluir]

                
                rutas_proxies_ebs = list({
                    "/".join(proxy.split("/")[:-1]) + "/" for proxy in proxies_ebs
                })
                proxy_seleccionado = ""
                
                if proxies_exp:
                    ubicacion_proxy = "/".join(proxies_exp[0].split("/")[:-1]) + "/"   # Carpeta (ubicación)
                    
                    ubicacion_proxy_ebs = "/".join(proxies_ebs[0].split("/")[:-1]) + "/"   # Carpeta (ubicación ebs)

                    st.markdown(
                        """
                        <div style="font-size:18px; font-weight:bold;">Proxy EXP</div>
                        """,
                        unsafe_allow_html=True
                    )

                    proxy_seleccionado = st.selectbox(
                        "Proxy EXP",
                        proxies_exp,
                        format_func=lambda x: x.split("/")[-1].rsplit(".", 1)[0],  # 👈 Solo muestra el nombre
                        label_visibility="collapsed"
                    )
                    
                    if proxy_seleccionado:
                        # Obtener el nombre de servicio del proxy
                        servicio = proxy_seleccionado.split("/")[-1].rsplit(".", 1)[0]
                        
                        pipeline_ref=""
                        ruta = proxy_seleccionado
                        st.session_state["ubicacion_proxy_exp"] = "/".join(ruta.split("/")[:-1]) + "/"   # Carpeta (ubicación)
                        servicio = ruta.split("/")[-1].rsplit(".", 1)[0] # Nombre del servicio (sin extensión)
                        
                        # Leer internamente el contenido del proxy dentro del JAR
                        with zipfile.ZipFile(jar_file, "r") as jar:
                            with jar.open(proxy_seleccionado) as proxy_file:
                                proxy_xml = proxy_file.read().decode("utf-8")

                                # Parsear XML
                                root = ET.fromstring(proxy_xml)

                                # Buscar el invoke con ref al pipeline
                                ns = {
                                    "ser": "http://www.bea.com/wli/sb/services",
                                    "con": "http://www.bea.com/wli/sb/pipeline/config"
                                }
                                st.session_state["ubicacion_pipeline_exp"] = None
                                invoke_elem = root.find(".//ser:invoke[@xsi:type='con:PipelineRef']", {
                                    **ns,
                                    "xsi": "http://www.w3.org/2001/XMLSchema-instance"
                                })
                                if invoke_elem is not None:
                                    pipeline_ref = invoke_elem.attrib.get("ref")

                                if pipeline_ref:
                                    ubicacion_pipeline_exp = pipeline_ref
                                    st.session_state["ubicacion_pipeline_exp"] = "/".join(ubicacion_pipeline_exp.split("/")[:-1]) + "/"
                                    st.markdown(f"📌 **Pipeline detectado:** `{pipeline_ref}`")
                                    st.session_state["ruta_pipeline_exp"] = pipeline_ref
                                else:
                                    st.warning("⚠️ No se encontró referencia a un Pipeline en este Proxy.")
                        
                        # Mostrar con subtítulo pequeño
                        pipeline_exp = pipeline_ref.split("/")[-1]
                        
                        st.markdown(
                            f"""
                            <div style="font-size:18px; font-weight:bold;">Nombre del servicio</div>
                            <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_proxy_exp"]}</div>
                            """,
                            unsafe_allow_html=True
                        )
                        
                        st.session_state["nombre_capa_exp"] = st.session_state["ubicacion_proxy_exp"].split("/")[0]

                        st.session_state["service_name"] = st.text_input(
                            "Nombre del servicio (interno)",
                            value=servicio,
                            disabled=True,
                            label_visibility="collapsed"
                        )
                        
                        st.markdown(
                            f"""
                            <div style="font-size:18px; font-weight:bold;">Nombre del pipeline</div>
                            <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_pipeline_exp"]}</div>
                            """,
                            unsafe_allow_html=True
                        )

                        st.session_state["pipeline_exp"] = st.text_input(
                            "Nombre del pipeline (interno)",
                            value=pipeline_exp,
                            disabled=True,
                            label_visibility="collapsed"
                        )
                        
                        # Obtener los WSDL asociados
                        wsdl_refs = obtener_wsdl_asociados(jar_file, proxy_seleccionado)
                        
                        st.session_state["ruta_wsdl"] = wsdl_refs
                        st.session_state["ubicacion_wsdl_exp"] = "/".join(st.session_state["ruta_wsdl"].split("/")[:-1]) + "/"   # Carpeta (ubicación)
                        st.session_state["wsdl_exp"] = st.session_state["ruta_wsdl"].split("/")[-1] # Nombre del servicio (sin extensión)
                        
                        
                        # Mostrar con subtítulo pequeño
                        st.markdown(
                            f"""
                            <div style="font-size:18px; font-weight:bold;">Nombre del wsdl</div>
                            <div style="font-size:12px; color:gray;">📂 {st.session_state["ubicacion_wsdl_exp"]}</div>
                            """,
                            unsafe_allow_html=True
                        )

                        st.session_state["wsdl_exp"] = st.text_input(
                            label="Nombre del wsdl",
                            value=st.session_state["wsdl_exp"],
                            disabled=True,
                            label_visibility="collapsed"
                        )
                        
                        ######################PARAMETROS INICIALES######################
                        
                        operation_name = st.text_input("Nombre de la operación", "")
                        
                        st.session_state["operation_name"] = operation_name.strip()
                        
                        if not st.session_state["operation_name"]:
                            st.warning("⚠ Digita el nombre de la operación.")
                        
                        # --------------------- NUEVO BLOQUE PARA CAPA EBS ---------------------
                        if st.session_state["operation_name"]:
                            with st.expander("⚙️ CAPA EBS"):
                                st.radio(
                                    "¿Requiere crear orquestado EBS?",
                                    options=["NO", "SI"],
                                    index=0,
                                    horizontal=True,
                                    key="requiere_ebs"
                                )

                                if st.session_state.get("requiere_ebs") == "SI":
                                    # proxy_seleccionado_ebs = st.selectbox(
                                        # "Proxy EBS",
                                        # proxies_ebs,
                                        # format_func=lambda x: x.split("/")[-1].rsplit(".", 1)[0],  # 👈 Solo muestra el nombre
                                        # label_visibility="collapsed"
                                    # )
                                    
                                    st.markdown(
                                            f"""
                                            <div style="font-size:18px; font-weight:bold;">Seleccione capa EBS</div>
                                            """,
                                            unsafe_allow_html=True
                                        )
                                    
                                    st.session_state["capa_seleccionada_ebs"] = st.selectbox(
                                            "Selecciona una ruta del proxy EBS:",
                                            rutas_proxies_ebs,
                                            format_func=lambda x: x.split("/")[0],  # 👈 muestra solo el nombre de la capa
                                            disabled=False,
                                            label_visibility="collapsed"
                                        )
                                    
                                    if st.session_state["capa_seleccionada_ebs"]:
                                        
                                        
                                        st.markdown(
                                            f"""
                                            <div style="font-size:18px; font-weight:bold;">Terminación EBS</div>
                                            """,
                                            unsafe_allow_html=True
                                        )
                                        st.session_state["terminacion_seleccionada_ebs"] = st.selectbox(
                                            "Terminación EBS:",
                                            terminacion_ebs,
                                            index=terminacion_ebs.index("AS") if "AS" in terminacion_ebs else 0,
                                            disabled=False,
                                            label_visibility="collapsed"
                                        )
                                        st.markdown(
                                            f"""
                                            <div style="font-size:18px; font-weight:bold;">Versión EBS</div>
                                            """,
                                            unsafe_allow_html=True
                                        )
                                        st.session_state["version_ebs"] = st.selectbox(
                                            "Versión EBS",
                                            options=["V1.0", "V1.1", "V1.2", "V2.0", "V2.1", "V2.2"],
                                            index=(
                                                ["V1.0", "V1.1", "V1.2", "V2.0", "V2.1", "V2.2"].index(st.session_state["version_ebs"])
                                                if "version_ebs" in st.session_state and st.session_state["version_ebs"] in ["V1.0", "V1.1", "V1.2", "V2.0", "V2.1", "V2.2"]
                                                else 0
                                            ),
                                            key="version_ebs_input",
                                            disabled=False,
                                            label_visibility="collapsed"
                                        )

                                        st.markdown(
                                            f"""
                                            <div style="font-size:18px; font-weight:bold;">Nombre del servicio EBS</div>
                                            <div style="font-size:12px; color:gray;">📂 {st.session_state["capa_seleccionada_ebs"]}</div>
                                            """,
                                            unsafe_allow_html=True
                                        )
                                        st.session_state["service_name_ebs"] = st.text_input(
                                            label="Nombre del servicio EBS",
                                            value=st.session_state["operation_name"]+st.session_state["terminacion_seleccionada_ebs"]+st.session_state["version_ebs"],
                                            disabled=True,
                                            label_visibility="collapsed"
                                        )
                                        
                                    
                            with st.expander("⚙️ CAPA ABC",expanded=True):
                                
                                st.radio(
                                    "¿Misma operacion ABC?",
                                    options=["SI", "NO"],
                                    index=0,
                                    horizontal=True,
                                    key="misma_operacion_abc"
                                )
                                
                                if st.session_state.get("misma_operacion_abc") == "NO":
                                    
                                    operation_name_abc = st.text_input("Nombre de la operación ABC", "")
                                    st.session_state["operation_name_abc"] = operation_name_abc.strip()
                                
                                else:
                                    st.session_state["operation_name_abc"] = st.text_input(
                                        label="Nombre de la operación ABC",
                                        value=st.session_state["operation_name"],
                                        disabled=True
                                    )
                                
                                st.session_state["nombre_capa_abc"] = st.text_input(
                                "Nombre capa ABC",
                                value=st.session_state["nombre_capa_abc"],  # recupera siempre
                                key="nombre_capa_abc_input")
                                
                                capa_abc_seleccionada = st.selectbox(
                                    "Carpeta ABC en el JAR",
                                    carpetas_raiz,
                                    key="capa_abc_seleccionada"
                                )
                                
                                
                                
                                st.session_state["version_proxy_abc"] = st.selectbox(
                                "Versión ABC",
                                options=["V2.1", "V1.0","V1.1", "V1.2", "V2.0","V2.2"],
                                index=(
                                    ["V2.1", "V1.0","V1.1", "V1.2", "V2.0","V2.2"].index(st.session_state["version_proxy_abc"])
                                    if "version_proxy_abc" in st.session_state and st.session_state["version_proxy_abc"] in ["V2.1", "V1.0","V1.1", "V1.2", "V2.0", "V2.2"]
                                    else 0
                                ),
                                key="version_proxy_abc_input")
                        
                else:
                    st.session_state["service_name"] = st.text_input(
                        "Nombre del servicio",
                        value="No se detecto Proxie EXP",
                        disabled=True
                    )

            if st.session_state["tipo_servicio"] == "Existente" and not jar_file:
                st.warning("⚠ Debes subir el archivo .jar para continuar.")
            
            if st.session_state["service_name"] == "No se detecto Proxie EXP":
                st.warning("⚠ Debes subir el archivo .jar que contenga el proxy EXP.")
            
            #################################FIN#################################
            
        else:
            st.session_state["service_name"] = st.text_input("Nombre del servicio expuesto (sin espacios)", "")

            # Lista de capas disponibles
            capas_disponibles = ["ABC", "EBS", "EXP"]

            # Multiselect para que el usuario elija una o varias capas
            capas_seleccionadas = st.multiselect(
                "Seleccione las capas que desea crear automáticamente:",
                capas_disponibles,
                default=["EXP", "ABC"]  # Puedes poner valores por defecto
            )
            
            if not st.session_state["service_name"]:
                st.warning("⚠ Digita el nombre del servicio.")
                
            # --- Inputs dinámicos para proyectos por capa seleccionada ---
            proyectos_por_capa = {}
            for capa in capas_seleccionadas:
                proyectos_por_capa[capa] = st.text_input(
                    f"Nombre del proyecto para capa {capa}", 
                    key=f"proyecto_{capa}"  # clave única en session_state
                )

            # (Opcional) Guardar en session_state para usarlos después
            st.session_state["proyectos_por_capa"] = proyectos_por_capa
            
            for capa in capas_seleccionadas:
                if capa == "EXP":
                    st.session_state["exp_proyecto"] = st.session_state["proyectos_por_capa"]["EXP"]
                elif capa == "EBS":
                    st.session_state["ebs_proyecto"] = st.session_state["proyectos_por_capa"]["EBS"]
                elif capa == "ABC":
                    st.session_state["nombre_capa_abc"] = st.session_state["proyectos_por_capa"]["ABC"]
                    
                if not st.session_state["proyectos_por_capa"][capa]:
                    st.warning(f"⚠ Digita el nombre de la capa {capa}")
                    
            operation_name = st.text_input("Nombre de la operación", "")
            
            # Siempre limpiar espacios en blanco antes de guardarlo
            st.session_state["operation_name"] = operation_name.strip()
            
            if not st.session_state["operation_name"]:
                st.warning("⚠ Digita el nombre de la operación.")
            else:
                st.session_state["operation_name_abc"] = st.session_state["operation_name"]
        
            st.session_state["version_proxy_abc"] = st.selectbox(
            "Versión ABC",
            options=["V1.0", "V1.1", "V1.2", "V2.0", "V2.1", "V2.2"],
            index=(
                ["V1.0", "V1.1", "V1.2", "V2.0", "V2.1", "V2.2"].index(st.session_state["version_proxy_abc"])
                if "version_proxy_abc" in st.session_state and st.session_state["version_proxy_abc"] in ["V1.0", "V1.1", "V1.2", "V2.0", "V2.1", "V2.2"]
                else 0
            ),
            key="version_proxy_input"
                )
        
            #################################FIN#################################

        if "generar_proyecto" not in st.session_state:
            st.session_state["generar_proyecto"] = False

        if st.button("Generar proyecto OSB"):
            st.session_state["generar_proyecto"] = True
            
            if st.session_state["service_name"] and st.session_state["operation_name"] and not st.session_state["nombre_capa_abc"]:
                st.warning("⚠ Digita el nombre de la capa ABC.")
         
    with st.container():
        
        if st.session_state["service_name"] and st.session_state["service_name"] != "No se detecto Proxie EXP" and st.session_state["nombre_capa_abc"] != "":
            generar_proyecto()
        

if __name__ == "__main__":
    main()