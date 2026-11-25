# SocialTech — Casa de José: escenario de simulación (mundo 3D + mapa 2D)

Paquete **solo de escenario**: un mundo de Gazebo Harmonic con la vivienda y su mapa de ocupación 2D.
**No incluye** ningún robot, ni software de localización, navegación o mapeo: cada equipo añade el suyo.

> **Geometría PROVISIONAL.** Las medidas salen de un plano en imagen con cotas que no casan entre sí. Se usó la escala gráfica del plano y valores supuestos
> (puertas de 0,90 m —la de entrada, de 1,00 m—, muros de 2,50 m, alturas de muebles). **No está validado con medidas reales.** Úsalo para pruebas de simulación,
> no como plano de la vivienda.

## Estructura
```text
worlds/casa_jose.sdf      mundo: suelo, muros, muebles y objetos de prueba (sistemas de física, sensores e IMU)
models/<modelo>/          un modelo por carpeta (model.config, model.sdf y, si la tiene, su malla)
maps/                     mapas de ocupación (PGM + YAML) y zonas prohibidas (polígonos)
rviz/casa_jose.rviz       vista de RViz2 del mapa
launch/casa_jose.launch.py  arranque conjunto Gazebo + RViz2 (ROS 2 Jazzy)
abrir_mundo.sh, ver_mapa.sh  arranque individual (Gazebo sin ROS 2 / mapa en RViz2)
package.xml, CMakeLists.txt  paquete ROS 2 opcional (colcon build); no es necesario para usar el escenario
```

## Requisitos
| | Versión probada | Notas |
|---|---|---|
| Ubuntu | 24.04 (Noble) | |
| **Gazebo Harmonic** | gz-sim **8.x** (probado con 8.15 / `gz sim`) | Imprescindible para el mundo 3D |
| ROS 2 **Jazzy** | opcional | Para el arranque conjunto (`launch/`), el mapa 2D en RViz2 y para conectar robots |

Instalación mínima (si no los tienes):
```bash
# Gazebo Harmonic (repositorio oficial de Gazebo)
sudo apt-get update && sudo apt-get install -y curl lsb-release gnupg
sudo curl https://packages.osrfoundation.org/gazebo.gpg --output /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null
sudo apt-get update && sudo apt-get install -y gz-harmonic
# (Opcional) si ya tienes ROS 2 Jazzy, para ver el mapa 2D:
sudo apt-get install -y ros-jazzy-ros-gz-sim ros-jazzy-nav2-map-server ros-jazzy-nav2-util ros-jazzy-rviz2 ros-jazzy-tf2-ros
```
(Con ROS 2 Jazzy y los paquetes `ros-jazzy-ros-gz`, Gazebo Harmonic ya viene incluido: `abrir_mundo.sh` carga `/opt/ros/jazzy/setup.bash` si `gz` no está en el PATH.)

## Uso
Las tres formas de arranque usan solo los archivos de esta carpeta y pueden ejecutarse desde cualquier directorio.

**1. Solo Gazebo (no necesita ROS 2)**
```bash
git clone https://github.com/alexllambrichcastells/socialtech-gazebo-worlds.git
cd socialtech-gazebo-worlds
./abrir_mundo.sh                 # con ventana;  ./abrir_mundo.sh --sin-gui  solo servidor
LIBGL_ALWAYS_SOFTWARE=1 ./abrir_mundo.sh     # máquinas virtuales sin aceleración 3D
```
A mano: `GZ_SIM_RESOURCE_PATH=$PWD/models gz sim -r worlds/casa_jose.sdf`.

**2. Gazebo y RViz2 con un solo comando (ROS 2 Jazzy)**
```bash
source /opt/ros/jazzy/setup.bash
ros2 launch launch/casa_jose.launch.py                       # Gazebo + RViz2 con el mapa
ros2 launch launch/casa_jose.launch.py rviz:=false           # solo Gazebo
ros2 launch launch/casa_jose.launch.py gui:=false            # servidor de Gazebo + RViz2
ros2 launch launch/casa_jose.launch.py map:=casa_jose_solo_muros
ros2 launch launch/casa_jose.launch.py --show-args          # todos los argumentos
```
Ctrl+C cierra todos los procesos; cerrar la ventana de Gazebo también. Cerrar RViz2 no detiene Gazebo (puede haber un robot en marcha).
El launch publica una TF estática identidad `world → map` (`tf_map:=false` para desactivarla si el robot ya publica `map`).

**3. Como paquete ROS 2 (opcional)**, para incluir el escenario desde el launch de un robot:
```bash
mkdir -p ~/ws/src && cd ~/ws/src && git clone https://github.com/alexllambrichcastells/socialtech-gazebo-worlds.git
cd ~/ws && colcon build && source install/setup.bash
ros2 launch socialtech_casa_jose casa_jose.launch.py
```
Al hacer `source` del workspace, `model://` resuelve los modelos sin configurar `GZ_SIM_RESOURCE_PATH`.
Desde otro launch: `get_package_share_directory('socialtech_casa_jose')` + `/worlds/casa_jose.sdf`.

**Solo el mapa 2D:** `./ver_mapa.sh [casa_jose | casa_jose_solo_muros | casa_jose_zonas_prohibidas]`.

## Añadir un robot
El mundo no contiene ningún robot. Con el mundo abierto, insertar el modelo propio (SDF o URDF) en la pose de inicio sugerida:
```bash
ros2 run ros_gz_sim create -world casa_jose -file <robot.sdf> -name mi_robot -x 6.597 -y 1.163 -z 0.05 -Y 0
# o sin ROS 2:
gz service -s /world/casa_jose/create --reqtype gz.msgs.EntityFactory --reptype gz.msgs.Boolean --timeout 5000 \
  --req 'sdf_filename: "<robot.sdf>", name: "mi_robot", pose: {position: {x: 6.597, y: 1.163, z: 0.05}}'
```
El mundo ya incluye los sistemas `Physics`, `Sensors` (render `ogre2`), `Imu`, `SceneBroadcaster` y `UserCommands`. Cada robot aporta sus propios
plugins (accionamiento, sensores), su puente `ros_gz_bridge` (`/clock`, `/scan`, `/cmd_vel`, `/odom`, `/tf`) y su TF. El escenario no publica `/clock` ni TF
(salvo `world → map` desde el launch).

Los sensores renderizados (`gpu_lidar`, cámaras) necesitan acceso a un servidor gráfico, también con `--sin-gui`/`gui:=false`. Sin él
(p. ej. por SSH o sin `XAUTHORITY`) los haces devuelven el alcance mínimo. En servidores sin pantalla: `xvfb-run` o un `DISPLAY` válido.

## Coordenadas
- El frame `map` de los mapas 2D coincide con el `world` de Gazebo (metros; z = 0 en el suelo).
- Mapas: resolución **0,01 m/píxel**, 1586 × 1422 píxeles (15,86 × 14,22 m), origen **(−0,1132, −0,0612)**.
- Pose de inicio sugerida: vestíbulo, frente a la puerta de entrada: **x = 6,597, y = 1,163, yaw = 0°** (`maps/zonas_prohibidas.yaml`).
- Zonas por las que un robot no debe pasar (ascensor, escaleras, puerta A de la vivienda vecina, patio): `maps/zonas_prohibidas.yaml` (polígonos) y
  `maps/casa_jose_zonas_prohibidas.pgm` (máscara; negro = prohibido).
- Pasos estrechos (supuestos): pasillo de ~0,885 m; puertas interiores ~0,88–0,90 m.

## Contenido del mundo
| Modelo | Descripción |
|---|---|
| `casa_jose_muros` | Muros de 2,50 m (malla STL en metros) |
| `banera`, `lavabo`, `inodoro`, `bide`, `bloque_cm` | Muebles dibujados en el plano (cajas; alturas supuestas) |
| `SINT_cnc_prueba_1`, `SINT_cnc_prueba_2` | **Objetos sintéticos de prueba** en el comedor; no existen en la vivienda |

| Mapa (`maps/`) | Contenido |
|---|---|
| `casa_jose` | muros + muebles + objetos de prueba (franja 0,05–1,80 m) |
| `casa_jose_solo_muros` | solo muros |
| `casa_jose_zonas_prohibidas` | máscara de zonas prohibidas |

## Limitaciones
- Geometría provisional (ver el aviso inicial); hojas de puerta no modeladas (huecos abiertos); ventanas modeladas como muro.
- Las rutas con espacios están soportadas en los scripts y en el launch (probado).
- Probado en Ubuntu 24.04 con ROS 2 Jazzy y Gazebo Harmonic 8 de los paquetes `ros-jazzy-ros-gz`, en una máquina virtual con render por software.
  No se ha probado con Gazebo Harmonic instalado sin ROS 2 (`gz-harmonic`).

## Licencia y origen
Todo el contenido (mundo, modelos, mallas y mapas) lo ha **generado el equipo SocialTech** para este ensayo a partir del plano de la vivienda.
**No incluye modelos ni texturas de terceros.** La malla de muros se generó con la herramienta pública *map2sdf* (Apache-2.0; su cabecera STL lo indica); el paquete no contiene su código. Representa una vivienda particular con **geometría aproximada**: se publica solo para pruebas de simulación; no es un plano de la vivienda.
Publicado con el consentimiento del propietario de la vivienda.

**Licencia: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.es)** (texto legal en `LICENSE`; créditos y alcance en `NOTICE`). Cita: *"Casa de José — escenario de simulación, Equipo SocialTech"*.
