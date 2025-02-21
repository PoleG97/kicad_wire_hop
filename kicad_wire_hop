import tkinter as tk
from tkinter import filedialog, messagebox
import pyperclip
import uuid
import re

def generar_uuid():
    """Genera un UUID en formato string."""
    return str(uuid.uuid4())

def contar_parentesis(line):
    """
    Devuelve (num_abiertos, num_cerrados) en la línea.
    Para ayudar a equilibrar bloques.
    """
    abiertos = line.count("(")
    cerrados = line.count(")")
    return abiertos, cerrados

def leer_bloque_desde(lines, start_index):
    """
    Lee desde 'start_index' un bloque que comienza con '('
    hasta que el número de paréntesis abiertos se equilibre con cerrados.
    Retorna:
      - bloque_texto (lista de líneas completas)
      - next_index (línea donde terminó el bloque)
    """
    bloque = []
    open_count = 0
    i = start_index
    bloque_iniciado = False

    while i < len(lines):
        line = lines[i]
        bloque.append(line)
        ab, ce = contar_parentesis(line)
        # Ajustamos contadores
        if not bloque_iniciado:
            # Asumimos que la línea start_index comienza con '('
            # Iniciamos la cuenta
            bloque_iniciado = True
            open_count = ab
            ce_count = ce
        else:
            open_count += ab
            ce_count += ce

        i += 1
        if open_count == ce_count and bloque_iniciado:
            # Se ha cerrado el bloque
            break

    return bloque, i

def es_wire_block(bloque_texto):
    """Devuelve True si el bloque empieza con '(wire '."""
    primer_line = bloque_texto[0].strip()
    return primer_line.startswith("(wire ")

def es_junction_block(bloque_texto):
    """Devuelve True si el bloque representa un junction."""
    # Normalmente KiCad lo escribe como (junction (at x y) ...)
    # Comprobamos la primera línea
    primer_line = bloque_texto[0].strip()
    return primer_line.startswith("(junction (at ")

def extraer_xy_pairs(bloque_texto):
    """
    Extrae las coordenadas (x,y) de (xy x_val y_val) dentro del bloque.
    Retorna lista de tuples [(x1,y1), (x2,y2), ...].
    """
    full_text = "\n".join(bloque_texto)
    pattern = r"\(xy\s+([-]?\d+\.?\d*)\s+([-]?\d+\.?\d*)\)"
    matches = re.findall(pattern, full_text)
    coords = []
    for mx, my in matches:
        coords.append((float(mx), float(my)))
    return coords

def extraer_junction_coord(bloque_texto):
    """
    Para un junction, extrae la coordenada (x,y).
    Normalmente en la forma: (junction (at X Y) (uuid "...") ...)
    """
    full_text = "\n".join(bloque_texto)
    pattern = r"\(at\s+([-]?\d+\.?\d*)\s+([-]?\d+\.?\d*)\)"
    m = re.search(pattern, full_text)
    if m:
        return (float(m.group(1)), float(m.group(2)))
    return None

def es_horizontal(coords):
    """
    Asume coords = [(x1,y1), (x2,y2), ...].
    Un wire KiCad normalmente tiene 2 puntos (a veces más).
    Para simplificar, consideramos un wire con 2 pts.
    Si más, hay que definir una lógica extra.
    """
    if len(coords) < 2:
        return False
    y0 = coords[0][1]
    for (x, y) in coords:
        if abs(y - y0) > 1e-9:
            return False
    return True

def es_vertical(coords):
    """
    Similar a es_horizontal, pero revisa si todos los x son iguales.
    """
    if len(coords) < 2:
        return False
    x0 = coords[0][0]
    for (x, y) in coords:
        if abs(x - x0) > 1e-9:
            return False
    return True

def intersecta_vert_horiz(xv, y_range, yh, x_range):
    """
    Dado un wire vertical en x=xv, y en [ymin,ymax],
    y un wire horizontal en y=yh, x en [xmin,xmax],
    mira si se cruzan en un punto interior (no extremo).
    Retorna x_int,y_int o None.
    """
    xmin, xmax = x_range
    ymin, ymax = y_range
    if xmin < xv < xmax and ymin < yh < ymax:
        return (xv, yh)
    return None

def generar_wire_line(x1, y1, x2, y2):
    return f"""(wire (pts (xy {x1:.2f} {y1:.2f}) (xy {x2:.2f} {y2:.2f}))
  (stroke (width 0) (type default))
  (uuid "{generar_uuid()}")
)"""

def generar_symbol_wirehop(xc, yc):
    """
    Genera un bloque de texto multiline para wireHop.
    Ajusta si quieres que se incluya en BOM o no, etc.
    """
    sym_uuid = generar_uuid()
    pin1_uuid = generar_uuid()
    pin2_uuid = generar_uuid()

    symbol_block = f"""(symbol (lib_id "wireHop:wireHop") (at {xc:.2f} {yc:.2f} 0) (unit 1)
  (exclude_from_sim no) (in_bom no) (on_board no) (fields_autoplaced yes)
  (uuid "{sym_uuid}")
  (property "Reference" "wp?" (at {xc:.2f} {yc-3.81:.2f} 0)
    (effects (font (size 1.27 1.27)) (hide yes))
  )
  (property "Value" "~" (at {xc:.2f} {yc-2.54:.2f} 0)
    (effects (font (size 1.27 1.27)))
  )
  (property "Footprint" "" (at {xc:.2f} {yc:.2f} 0)
    (effects (font (size 1.27 1.27)) (hide yes))
  )
  (property "Datasheet" "" (at {xc:.2f} {yc:.2f} 0)
    (effects (font (size 1.27 1.27)) (hide yes))
  )
  (property "Description" "" (at {xc:.2f} {yc:.2f} 0)
    (effects (font (size 1.27 1.27)) (hide yes))
  )
  (pin "1" (uuid "{pin1_uuid}"))
  (pin "1" (uuid "{pin2_uuid}"))
)"""
    return symbol_block.splitlines()

def procesar_bloques(blocks):
    """
    - Detecta wires verticales y horizontales
    - Encuentra cruces
    - Corta wires horizontales en sub-tramos
    - Inserta wireHop
    - Devuelve la lista final de bloques (texto) con cambios
    """
    wires_info = []
    junctions = set()

    for i, (tipo, texto, coords) in enumerate(blocks):
        if tipo == "wire":
            wires_info.append((i, coords))
        elif tipo == "junction":
            if coords is not None:
                junctions.add(coords)

    cruces_por_wireH = {}
    wire_data = {}
    for (idx, coords) in wires_info:
        if len(coords) >= 2:
            xvals = [c[0] for c in coords]
            yvals = [c[1] for c in coords]
            xmin, xmax = sorted(xvals)
            ymin, ymax = sorted(yvals)
            wire_data[idx] = {
                'coords': coords,
                'xmin': xmin,
                'xmax': xmax,
                'ymin': ymin,
                'ymax': ymax,
                'is_horizontal': es_horizontal(coords),
                'is_vertical': es_vertical(coords)
            }

    w_indices = list(wire_data.keys())
    for i in range(len(w_indices)):
        idxA = w_indices[i]
        dA = wire_data[idxA]
        for j in range(i+1, len(w_indices)):
            idxB = w_indices[j]
            dB = wire_data[idxB]
            if dA['is_vertical'] and dB['is_horizontal']:
                xv = dA['coords'][0][0]
                yh = dB['coords'][0][1]
                p = intersecta_vert_horiz(
                    xv,
                    (dA['ymin'], dA['ymax']),
                    yh,
                    (dB['xmin'], dB['xmax'])
                )
                if p:
                    if p not in junctions:
                        if idxB not in cruces_por_wireH:
                            cruces_por_wireH[idxB] = []
                        cruces_por_wireH[idxB].append(p[0])
            elif dA['is_horizontal'] and dB['is_vertical']:
                xv = dB['coords'][0][0]
                yh = dA['coords'][0][1]
                p = intersecta_vert_horiz(
                    xv,
                    (dB['ymin'], dB['ymax']),
                    yh,
                    (dA['xmin'], dA['xmax'])
                )
                if p:
                    if p not in junctions:
                        if idxA not in cruces_por_wireH:
                            cruces_por_wireH[idxA] = []
                        cruces_por_wireH[idxA].append(p[0])

    offset = 3.81
    nuevos_blocks = []

    for i, (tipo, texto, coords) in enumerate(blocks):
        if tipo != "wire":
            nuevos_blocks.append((tipo, texto, coords))
            continue

        if i not in cruces_por_wireH:
            nuevos_blocks.append((tipo, texto, coords))
            continue

        xvals = [c[0] for c in coords]
        yvals = [c[1] for c in coords]
        xmin, xmax = sorted(xvals)
        yh = yvals[0]
        x_cruces = sorted(set(cruces_por_wireH[i]))
        x_cruces_validos = [xc for xc in x_cruces if xmin < xc < xmax]

        if len(x_cruces_validos) == 0:
            nuevos_blocks.append((tipo, texto, coords))
            continue

        x_inicio = xmin
        for xc in x_cruces_validos:
            x_left = xc - offset
            if x_left < x_inicio:
                x_left = x_inicio
            if x_left > x_inicio:
                wire_txt = generar_wire_line(x_inicio, yh, x_left, yh).splitlines()
                nuevos_blocks.append(("wire", wire_txt, [(x_inicio, yh), (x_left, yh)]))
            hop_txt = generar_symbol_wirehop(xc, yh)
            nuevos_blocks.append(("symbol", hop_txt, None))
            x_inicio = xc + offset

        if x_inicio < xmax:
            wire_txt = generar_wire_line(x_inicio, yh, xmax, yh).splitlines()
            nuevos_blocks.append(("wire", wire_txt, [(x_inicio, yh), (xmax, yh)]))

    return nuevos_blocks

def rearmar_sch(blocks):
    """
    Convierte la lista de bloques de vuelta a texto.
    Cada bloque es (tipo, [lineas_de_texto], coords_opcionales).
    """
    output_lines = []
    for (tipo, texto, coords) in blocks:
        output_lines.extend(texto)
    return "\n".join(output_lines)

def procesar_sch(contenido):
    """
    - Separa el .kicad_sch en bloques (wire, junction, otros)
    - Realiza la lógica de cruces
    - Retorna el .kicad_sch modificado como string
    """
    lines = contenido.splitlines()
    i = 0
    n = len(lines)
    blocks = []

    while i < n:
        line = lines[i].rstrip("\n")
        stripped = line.strip()

        if stripped.startswith("(wire "):
            wire_block, next_i = leer_bloque_desde(lines, i)
            coords = extraer_xy_pairs(wire_block)
            blocks.append(("wire", wire_block, coords))
            i = next_i
        elif stripped.startswith("(junction (at "):
            j_block, next_i = leer_bloque_desde(lines, i)
            j_coord = extraer_junction_coord(j_block)
            blocks.append(("junction", j_block, j_coord))
            i = next_i
        else:
            if stripped.startswith("("):
                gen_block, next_i = leer_bloque_desde(lines, i)
                blocks.append(("other", gen_block, None))
                i = next_i
            else:
                blocks.append(("other", [line], None))
                i += 1

    bloques_modif = procesar_bloques(blocks)
    return rearmar_sch(bloques_modif)

# ---------------------------------------------------------
# Interfaz Tkinter para usar el script
# ---------------------------------------------------------

class WireHopApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Conversor de cruces de cables a wireHop")

        self.frame = tk.Frame(root)
        self.frame.pack(padx=10, pady=10, fill="both", expand=True)

        self.modo_var = tk.StringVar(value="pegar")
        tk.Radiobutton(self.frame, text="Pegar contenido SCH", variable=self.modo_var, value="pegar",
                       command=self.actualizar_modo).grid(row=0, column=0, sticky="w")
        tk.Radiobutton(self.frame, text="Cargar archivo SCH", variable=self.modo_var, value="archivo",
                       command=self.actualizar_modo).grid(row=0, column=1, sticky="w")

        self.texto_label = tk.Label(self.frame, text="Contenido SCH:")
        self.texto_label.grid(row=1, column=0, columnspan=2, sticky="w")

        self.text_sch = tk.Text(self.frame, width=80, height=15)
        self.text_sch.grid(row=2, column=0, columnspan=2, padx=5, pady=5)

        self.btn_cargar = tk.Button(self.frame, text="Cargar Archivo...", command=self.cargar_archivo)
        self.btn_cargar.grid(row=3, column=0, sticky="w")

        self.btn_convertir = tk.Button(self.frame, text="Convertir", command=self.convertir)
        self.btn_convertir.grid(row=3, column=1, sticky="e")

        self.label_resultado = tk.Label(self.frame, text="Resultado:")
        self.label_resultado.grid(row=4, column=0, columnspan=2, sticky="w")

        self.text_resultado = tk.Text(self.frame, width=80, height=15)
        self.text_resultado.grid(row=5, column=0, columnspan=2, padx=5, pady=5)

        self.btn_copiar = tk.Button(self.frame, text="Copiar al portapapeles", command=self.copiar_portapapeles)
        self.btn_copiar.grid(row=6, column=0, columnspan=2, pady=5)

        # Nuevo frame para los botones de borrado
        self.clear_frame = tk.Frame(self.frame)
        self.clear_frame.grid(row=7, column=0, columnspan=2, pady=5)

        self.btn_clear_input = tk.Button(self.clear_frame, text="Borrar entrada", command=self.clear_input)
        self.btn_clear_input.pack(side="left", padx=5)

        self.btn_clear_output = tk.Button(self.clear_frame, text="Borrar salida", command=self.clear_output)
        self.btn_clear_output.pack(side="left", padx=5)

        self.btn_clear_both = tk.Button(self.clear_frame, text="Borrar ambas", command=self.clear_both)
        self.btn_clear_both.pack(side="left", padx=5)

        self.actualizar_modo()

    def actualizar_modo(self):
        modo = self.modo_var.get()
        if modo == "pegar":
            self.btn_cargar.config(state="disabled")
            self.text_sch.config(state="normal")
        else:
            self.btn_cargar.config(state="normal")
            self.text_sch.config(state="disabled")

    def cargar_archivo(self):
        ruta = filedialog.askopenfilename(title="Selecciona un archivo .kicad_sch", filetypes=[("KiCad SCH", "*.kicad_sch")])
        if ruta:
            with open(ruta, "r", encoding="utf-8") as f:
                contenido = f.read()
            self.text_sch.config(state="normal")
            self.text_sch.delete("1.0", tk.END)
            self.text_sch.insert(tk.END, contenido)
            self.text_sch.config(state="disabled")
            self.modo_var.set("archivo")

    def convertir(self):
        modo = self.modo_var.get()
        if modo == "pegar":
            contenido = self.text_sch.get("1.0", tk.END)
        else:
            contenido = self.text_sch.get("1.0", tk.END)

        if not contenido.strip():
            messagebox.showwarning("Aviso", "No hay contenido para procesar.")
            return

        resultado = procesar_sch(contenido)
        self.text_resultado.delete("1.0", tk.END)
        self.text_resultado.insert(tk.END, resultado)

    def copiar_portapapeles(self):
        texto = self.text_resultado.get("1.0", tk.END)
        pyperclip.copy(texto)
        messagebox.showinfo("Copiado", "El contenido convertido se ha copiado al portapapeles.")

    # Métodos para borrar textos
    def clear_input(self):
        self.text_sch.config(state="normal")
        self.text_sch.delete("1.0", tk.END)
        if self.modo_var.get() == "archivo":
            self.text_sch.config(state="disabled")

    def clear_output(self):
        self.text_resultado.delete("1.0", tk.END)

    def clear_both(self):
        self.clear_input()
        self.clear_output()

if __name__ == "__main__":
    root = tk.Tk()
    app = WireHopApp(root)
    root.mainloop()
