## 1. Clonar Repositorio de `minsar`
Repositorio de la organización InSAR:
```
https://github.com/finsarlab/minsar-cluster
```

Se debe clonar el repositorio via `ssh`:
```
git clone git@github.com:finsarlab/minsar-cluster.git
```

Se puede generar una llave ssh específica, pero se puede usar la que ya existe dado el procedimiento correcto de [[Uso de Clúster ADMIN#Añadir usuarios al Cluster|creación de usuario]]
```sh
[monkee@master ~]$ ls ~/.ssh  
authorized_keys  id_ed25519  id_ed25519.pub  known_hosts
```
### Añadir la llave:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

add minsar_home alias to .bashrc


copiar a .bashrc :

	export MINSAR_HOME=~/minsar

	shopt -s expand_aliases

	alias s.bw2='source $MINSAR_HOME/setup/environment.bash; conda activate minsar'


copiar el directorio accounts a carpeta accounts dentro del usuario

el script de install_minsar.bash tiene un error, le falta shebang al script

comentar linea 8, 19, 20, 24

dentro del directorio minsar correr:
	set -eo pipefail
./setup/install_python.bash
./setup/install_minsar.bash (Si falla, repetirlo)

./setup/install_credential_files.bash (dara un error, pero no importa)


modificar archivo setup/platforms_defaults.bash, copiar uno ya existente con los cambios para mayor facilidad

./setup/setup_orbit_dirs.bash



gmt
xarray
numpy
pandas
netcdf4