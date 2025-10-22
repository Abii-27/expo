<?php
error_reporting(E_ALL);
ini_set('display_errors', 1);

// 🔑 Conexión a Supabase
$host = "aws-1-us-east-2.pooler.supabase.com";
$port = "5432";
$dbname = "postgres";
$user = "postgres.orzsdjjmyouhhxjfnemt";
$password = "Zv2sW23OhBVM5Tkz";

try {
    $conn = new PDO("pgsql:host=$host;port=$port;dbname=$dbname", $user, $password);
    $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    if ($_SERVER["REQUEST_METHOD"] == "POST") {
        // 📷 Manejo de imagen
        $imagenPath = null;
        if (isset($_FILES['imagen']) && $_FILES['imagen']['error'] == 0) {
            $directorio = "uploads/";
            if (!is_dir($directorio)) {
                mkdir($directorio, 0777, true);
            }
            $nombreArchivo = uniqid() . "_" . basename($_FILES["imagen"]["name"]);
            $rutaDestino = $directorio . $nombreArchivo;

            if (move_uploaded_file($_FILES["imagen"]["tmp_name"], $rutaDestino)) {
                $imagenPath = $rutaDestino;
            }
        }

        // 📋 Datos del formulario
        $nombre_completo = $_POST['nombre_completo'] ?? null;
        $fecha_nacimiento = $_POST['fecha_nacimiento'] ?? null;
        $dui = $_POST['dui'] ?? null;

        if (isset($_POST['sexo']) && $_POST['sexo'] === "1") {
            $sexo = true;
        } elseif (isset($_POST['sexo']) && $_POST['sexo'] === "0") {
            $sexo = false;
        } else {
            $sexo = null;
        }

        $profesion = $_POST['profesion'] ?? null;
        $telefono = $_POST['telefono'] ?? null;
        $correo = $_POST['correo'] ?? null;
        $direccion = $_POST['direccion'] ?? null;
        $linkedin = $_POST['linkedin'] ?? null;
        $resumen_profesional = $_POST['resumen_profesional'] ?? null;
        $experiencia = $_POST['experiencia'] ?? null;
        $educacion = $_POST['educacion'] ?? null;
        $habilidades = $_POST['habilidades'] ?? null;
        $idiomas = $_POST['idiomas'] ?? null;
        $certificaciones = $_POST['certificaciones'] ?? null;
        $cursos = $_POST['cursos'] ?? null;
        $salario_pretendido = $_POST['salario_pretendido'] ?: null;

        // 🚀 Inserción SQL
        $sql = "INSERT INTO curriculum (
            imagen, nombre_completo, fecha_nacimiento, dui, sexo, profesion, telefono, correo, direccion, linkedin,
            resumen_profesional, experiencia, educacion, habilidades, idiomas, certificaciones, cursos, salario_pretendido
            ) VALUES (
                :imagen, :nombre_completo, :fecha_nacimiento, :dui, :sexo, :profesion, :telefono, :correo, :direccion, :linkedin,
                :resumen_profesional, :experiencia, :educacion, :habilidades, :idiomas, :certificaciones, :cursos, :salario_pretendido
                )";

        $stmt = $conn->prepare($sql);

        $stmt->bindValue(':imagen', $imagenPath);
        $stmt->bindValue(':nombre_completo', $nombre_completo);
        $stmt->bindValue(':fecha_nacimiento', $fecha_nacimiento);
        $stmt->bindValue(':dui', $dui);
        $stmt->bindValue(':sexo', $sexo, $sexo === null ? PDO::PARAM_NULL : PDO::PARAM_BOOL);
        $stmt->bindValue(':profesion', $profesion);
        $stmt->bindValue(':telefono', $telefono);
        $stmt->bindValue(':correo', $correo);
        $stmt->bindValue(':direccion', $direccion);
        $stmt->bindValue(':linkedin', $linkedin);
        $stmt->bindValue(':resumen_profesional', $resumen_profesional);
        $stmt->bindValue(':experiencia', $experiencia);
        $stmt->bindValue(':educacion', $educacion);
        $stmt->bindValue(':habilidades', $habilidades);
        $stmt->bindValue(':idiomas', $idiomas);
        $stmt->bindValue(':certificaciones', $certificaciones);
        $stmt->bindValue(':cursos', $cursos);
        $stmt->bindValue(':salario_pretendido', $salario_pretendido, $salario_pretendido === null ? PDO::PARAM_NULL : PDO::PARAM_STR);

        $stmt->execute();

        // ✅ Generar PDF
        require('fpdf/fpdf.php');
        $pdf = new FPDF();
        $pdf->AddPage();
        $pdf->SetMargins(15,15,10);

        $pdf->SetFillColor(40, 51, 110);
        $pdf->SetTextColor(255,255,255);
        $pdf->SetFont('Arial','B',20);
        $pdf->Cell(0,12, iconv("UTF-8","ISO-8859-1//TRANSLIT",'Curriculum'),0,1,'C', true);
        $pdf->Ln(5);

     // --- Agregar imagen al PDF (verificada/conversión segura + depuración) ---
if ($imagenPath) {
    // ✅ Asegurar ruta absoluta correcta (para que FPDF la encuentre)
    $imagenPath = __DIR__ . '/uploads/' . basename($imagenPath);
    $imagenAbsoluta = realpath($imagenPath);

    // 🧾 Mostrar en PDF la ruta usada (debug)
    $pdf->SetFont('Arial', 'I', 8);
    $pdf->SetTextColor(100, 100, 100);
    $pdf->Cell(0, 8, 'Ruta imagen: ' . ($imagenAbsoluta ?: 'No encontrada'), 0, 1);

    if ($imagenAbsoluta && file_exists($imagenAbsoluta)) {
        $info = getimagesize($imagenAbsoluta);
        if ($info !== false) {
            $mime = $info['mime'];
            $tempFile = $imagenAbsoluta;

            // Si no es JPG real, convertir temporalmente
            if ($mime !== 'image/jpeg') {
                $img = null;
                switch ($mime) {
                    case 'image/png':
                        $img = imagecreatefrompng($imagenAbsoluta);
                        break;
                    case 'image/gif':
                        $img = imagecreatefromgif($imagenAbsoluta);
                        break;
                    case 'image/webp':
                        $img = imagecreatefromwebp($imagenAbsoluta);
                        break;
                }
                if ($img) {
                    $tempFile = sys_get_temp_dir() . '/tmp_' . uniqid() . '.jpg';
                    imagejpeg($img, $tempFile, 90);
                    imagedestroy($img);
                }
            }

            // ✅ Mostrar la imagen convertida o original
            if (file_exists($tempFile)) {
                $pdf->Image($tempFile, 150, 20, 40);
            } else {
                $pdf->SetFont('Arial', 'I', 10);
                $pdf->SetTextColor(255, 0, 0);
                $pdf->Cell(0, 10, 'No se pudo cargar la imagen (archivo temporal no existe).', 0, 1);
            }
        } else {
            $pdf->SetFont('Arial', 'I', 10);
            $pdf->SetTextColor(255, 0, 0);
            $pdf->Cell(0, 10, 'Error: no se pudo leer info de la imagen.', 0, 1);
        }
    } else {
        $pdf->SetFont('Arial', 'I', 10);
        $pdf->SetTextColor(255, 0, 0);
        $pdf->Cell(0, 10, 'Error: no se encontró la imagen en la ruta indicada.', 0, 1);
    }
}

        $pdf->SetFillColor(230,230,250);
        $pdf->SetTextColor(0,0,0);
        $pdf->SetFont('Arial','B',14);
        $pdf->Cell(140,10, iconv("UTF-8","ISO-8859-1//TRANSLIT",'Datos Personales'),0,1,'L', true);
        $pdf->Ln(2);
        $pdf->SetFont('Arial','',12);

        $campos = [
            "Nombre completo" => $nombre_completo,
            "Fecha de nacimiento" => $fecha_nacimiento,
            "DUI" => $dui,
            "Sexo" => ($sexo === true ? "Masculino" : ($sexo === false ? "Femenino" : "No definido")),
            "Profesión" => $profesion,
            "Teléfono" => $telefono,
            "Correo" => $correo,
            "Dirección" => $direccion,
            "LinkedIn" => $linkedin
        ];
        foreach ($campos as $label => $valor) {
            $pdf->Cell(50,8, iconv("UTF-8","ISO-8859-1//TRANSLIT","$label: "));
            $pdf->Cell(0,8, iconv("UTF-8","ISO-8859-1//TRANSLIT",$valor),0,1);
        }

        $secciones = [
            "Resumen Profesional" => $resumen_profesional,
            "Experiencia" => $experiencia,
            "Educación" => $educacion,
            "Habilidades" => $habilidades,
            "Idiomas" => $idiomas,
            "Certificaciones" => $certificaciones,
            "Cursos" => $cursos
        ];
        foreach ($secciones as $titulo => $contenido) {
            $pdf->Ln(3);
            $pdf->SetFont('Arial','B',14);
            $pdf->Cell(0,10, iconv("UTF-8","ISO-8859-1//TRANSLIT",$titulo),0,1,'L', true);
            $pdf->SetFont('Arial','',12);
            $pdf->MultiCell(0,8, iconv("UTF-8","ISO-8859-1//TRANSLIT",$contenido));
        }

        $pdf->Ln(2);
        $pdf->SetFont('Arial','B',14);
        $pdf->Cell(0,10, iconv("UTF-8","ISO-8859-1//TRANSLIT",'Salario Pretendido: '),0,1,'L', true);
        $pdf->Cell(0,8, iconv("UTF-8","ISO-8859-1//TRANSLIT",$salario_pretendido),0,1);

        $pdf->Output('D','Hoja_de_Vida_'.iconv("UTF-8","ISO-8859-1//TRANSLIT",$nombre_completo).'.pdf');
        exit;
    }
} catch (PDOException $e) {
    echo "❌ Error: " . $e->getMessage();
}
?>

<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<title>Registro de Empleados</title>
<style>
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 0;
    background: #0b2a56;
    color: #333;
}
h2 { text-align:center; color:#fff; padding:20px 0 10px; margin:0; font-size:26px; }
.nav {
    display:flex; justify-content:flex-start; gap:15px; background:#0b2a56; padding:12px 20px; position:sticky; top:0;
}
.nav .btn {
    background:#1e3c72; color:#fff; padding:10px 18px; border-radius:6px; border:none; cursor:pointer; font-size:15px; font-weight:bold; transition: background 0.3s ease, transform 0.2s ease;
}
nav .btn:hover { background:#2a5298; transform:scale(1.05); }
form {
    max-width:1000px; margin:30px auto; padding:30px; border-radius:12px; background:#fff; box-shadow:0px 6px 15px rgba(0,0,0,0.2);
    display:grid; grid-template-columns:1fr 1fr; gap:20px 25px;
}
label { font-weight:bold; color:#1e3c72; margin-bottom:6px; display:block; }
input, textarea, select {
    width:100%; padding:8px; border:1px solid #ccc; border-radius:6px;
    font-size:14px; transition:border 0.3s ease, box-shadow 0.3s ease;
}
input:focus, textarea:focus, select:focus {
    border-color:#2a5298; outline:none; box-shadow:0 0 5px rgba(42,82,152,0.5);
}
textarea { resize:vertical; min-height:60px; }
.small-input { max-width:200px; }
button[type="submit"] {
    grid-column:span 2; padding:14px;
    background:#1e3c72; color:white;
    font-size:16px; font-weight:bold;
    border:none; border-radius:6px;
    cursor:pointer; transition: background 0.3s ease, transform 0.2s ease;
}
button[type="submit"]:hover { background:#2a5298; transform:scale(1.03); }
</style>
</head>
<body>
    <div class="nav">
<button class="btn" onclick="window.location.href='http://localhost/login.php'">Login</button>
<button class="btn" onclick="window.location.href='http://localhost/empleados_index.php'">Empleados</button>
<button class="btn" onclick="window.location.href='http://localhost/api.php'">Asistencia</button>
<button class="btn" onclick="window.location.href='http://localhost/formulario.php'">Registrar</button>
</div>

<h2>Registro de Empleados</h2>
<form action="" method="POST" enctype="multipart/form-data" autocomplete="off">
<div><label>Imagen</label><input type="file" name="imagen" accept="image/*"></div>
<div><label>Nombre Completo</label><input type="text" name="nombre_completo" required></div>
<div><label>Fecha de Nacimiento</label><input type="date" name="fecha_nacimiento"></div>
<div><label>DUI</label><input type="text" name="dui" maxlength="10" class="small-input"></div>
<div><label>Sexo</label>
<select name="sexo">
<option value="">-- Seleccione --</option>
<option value="1">Masculino</option>
<option value="0">Femenino</option>
</select>
</div>
<div><label>Profesión</label><input type="text" name="profesion"></div>
<div><label>Teléfono</label><input type="text" name="telefono" maxlength="8" class="small-input"></div>
<div><label>Correo Electrónico</label><input type="email" name="correo" required></div>
<div><label>Dirección</label><textarea name="direccion"></textarea></div>
<div><label>LinkedIn</label><input type="url" name="linkedin"></div>
<div><label>Resumen Profesional</label><textarea name="resumen_profesional"></textarea></div>
<div><label>Experiencia Laboral</label><textarea name="experiencia"></textarea></div>
<div><label>Educación</label><textarea name="educacion"></textarea></div>
<div><label>Habilidades</label><textarea name="habilidades"></textarea></div>
<div><label>Idiomas</label><input type="text" name="idiomas" class="small-input"></div>
<div><label>Certificaciones</label><textarea name="certificaciones"></textarea></div>
<div><label>Cursos</label><textarea name="cursos"></textarea></div>
<div><label>Salario Pretendido (USD)</label><input type="number" step="0.01" name="salario_pretendido" class="small-input"></div>
<button type="submit">Registrar nuevo empleado</button>
</form>
</body>
</html>
# expo
