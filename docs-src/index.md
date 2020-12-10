# Compilando Android numa Raspbery Pi 3 

- **Alunes:** Emanuelle Moço / Leonardo Mendes / Lucas Leal
- **Curso:** Engenharia da Computação
- **Semestre:** 6
- **Contato:** corsiferrao@gmail.com
- **Ano:** 2020

## Começando

Para seguir esse tutorial é necessário:

- **Hardware** 
    - Raspberry Pi 3 
    - Micro SD Card de 8 GB ou maior
    - Monitor com entrada   
    - Teclado e Mouse USB
    - Adaptador Micro SD para USB 
    - Cabo Micro USB
    - Computador com pelo menos 150GB de armazenamento disponível   
- **Softwares** 
    - Ubuntu 18.04
- **Referências:** [device_brcm_rpi3](https://github.com/android-rpi/device_brcm_rpi3), [Build Android for Raspberry Pi3](https://github.com/tab-pi/platform_manifest), [Building Android for Raspberry Pi](https://github.com/csimmonds/a4rpi-local-manifest/)
    

## Motivação  
A motivação incial era compilar Android em uma FPGA e o tema foi escolhido pois o grupo tinha o desejo de compilar um software de alto nível em uma placa de desenvolvimento. Entretanto, por dificuldades tecnicas, foi decidido migrar para uma Raspberry Pi.

----------------------------------------------

## Contexto 

O sistema operacional android é extremamente popular e muito presente em disposítivos mobile. Este tutorial consiste em embarcar o Android utilizando LineageOS, uma distribuição opensource de Android, numa Raspberry Pi 3 B+. 

### Android

Android é um sistema operacional baseado no kernel do linux no qual é possível compilar o seu próprio sistema Android para utilizar em um celular ou placas de desenvolvimento.  

### Raspberry Pi 

Raspberry Pi é um _microcomputador_ do estilo _System On a Chip_ que permite fazer tudo que um computador faz com baixo custo.
O modelo utilizado neste roteiro é o Raspberry Pi 3 B+, a qual possui um processador 64-bit quad-core de 1.4GHz, dual-band wireless e bluetooth. Ou seja, adequada para aplicações de automação e IOT.

<center>![Raspberry PI3](https://uploads.filipeflop.com/2017/07/DRA01_01.jpg){width=300}</center>

----------------------------------------------

## Instalação

!!! warning
    Alguns dos passos exigem alto poder computacional, caso você não tenha uma máquina com o armazenamento mínimo necessário ou com bom processador, sugerimos a utilização de uma máquina na nuvem.
    
    !!! example "Seguestão"
        Para esse tutorial, utilizamos uma instância da AWS t2.2xlarge.


### Configurando o ambiente
Uma vez no Ubuntu 18.04, será necessário a instalação de alguns pacotes essênciais, para saber mais, entre na página disponibilizada pelo [Andoid](https://source.android.com/setup/build/initializing).
```sh
$ sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig openjdk-8-jdk gcc-arm-linux-gnueabihf libssl-dev python-mako
```
Instale também o [repo](https://source.android.com/setup/develop#repo):
```
$ repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r25
```

Clone o reposiório com as configurações da Raspberry para Android:

```
$ git clone https://github.com/csimmonds/a4rpi-local-manifest .repo/local_manifests -b android10
```

!!! Nota

     Para aumentar a velocidade de instalação use o argumento -c (branch atual) e -j```threadcount``` 
```
$ repo sync -c j8
```

!!! warning
    Pausa para café, esta etapa demora um pouco.

### Configurando o U-boot

U-boot é um bootloader Opensource utilizado em sistemas de linux embarcados. Os comandos abaixo criam a nossa imagem _boot_ que será utilizada para carregar o Android.
```
$ cd $ANDROID_BUILD_TOP/u-boot
$ PATH=$HOME/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf/bin:$PATH
$ export ARCH=arm
$ export CROSS_COMPILE=arm-linux-gnueabihf-
$ cp $ANDROID_BUILD_TOP/device/rpiorg/rpi3/u-boot/rpi_3_32b_android_defconfig configs
$ make rpi_3_32b_android_defconfig
$ make

```

### Compilando o kernel

É necessário compilar o Kernel que será responsável pela criação do Android. Também é criada a partição DTBS, _device tree blob source_, que é responsável por disponibilizar a estrutura do hardware. Como Android pode ser utilizado dispositivos diferentes, Device Tree Overlays (DTOs) são necessários para mapear o hardware para o sistema.  


Rode os comandos: 
``` 
$ PATH=$HOME/gcc-linaro-7.4.1-2019.02-x86_64_arm-linux-gnueabihf/bin:$PATH
$ export ARCH=arm
$ export CROSS_COMPILE=arm-linux-gnueabihf-
$ cd $ANDROID_BUILD_TOP/kernel/rpi
$ scripts/kconfig/merge_config.sh arch/arm/configs/bcm2709_defconfig \
kernel/configs/android-base.config kernel/configs/android-recommended.config
$ make -j $(nproc) zImage
$ cp arch/arm/boot/zImage $ANDROID_BUILD_TOP/device/rpiorg/rpi3
$ make dtbs
$ croot
```

### Compilando o Android

Com as imagens criadas anteriormente (zImage, boot e dtbs), agora temos tudo pronto para compilar o Andorid em si. Para isso, é necessário configurar as variáveis de ambiente e fazer a montagem.  

```
$ source build/envsetup.sh
$ lunch aosp_rpi3-eng
$ m
```
!!! warning
    Esta etapa pode demorar em torno de 2 horas.

A explicação dos comandos utilizadas pode ser encontrada [aqui](https://source.android.com/setup/build/building).  É criado as imagens VendorImage, SystemImage e UserData que, posteriormente serão escritas no cartão de memória. 

---------------------------------------------- 

### Escrevendo no cartão SD 

A última etapa é  criar as partições e passar as imagens criadas no passo anterior para partições no SD card, que são elas:

- Vendor:  Drivers que conectam hardware e software;
- System: Sistema Android;
- UserData: Usado para resetar as configurações;
- Boot: Arquivos de inicialização, como os DTOs etc e configurações do Uboot.



!!! warning
    Caso você tenha feito as etapas anteriores em uma instância virtual, siga os passos abaixo.

??? Nuvem 

    Como não é possível conectar um cartão SD diretamente em uma instância na nuvem, deve ser criado uma partição virtual para simular um cartão SD. 
    
    Primeiro, é necessário criar uma imagem vazia com o tamanho disponível do seu cartão SD.

    ```sh
    dd if=/dev/zero of=zero.img bs=4M count=1536
    ```
    Com isso, criaremos a partição virtual a partir dessa imagem:
   
    ```sh
    $ sudo losetup -P -f --show zero.img
    ```
     Ao término do comando, a sua saída será o nome da sua partição virtual.  

     Modifique o arquivo /aosp/scripts/write-sdcard-rpi3.sh, alterando **_mmcblk0_** para o nome da sua partição




Use comando _lsblk_ para saber o nome do dispositivo no seu computador. 
No exemplo abaixo, o nome do dispositivo é _sdc_.

<center>![](https://media.discordapp.net/attachments/727592935054639194/786344627417382912/unknown.png){width=300}</center>

Agora, para instalar o Android no SD Card é necessário rodar o comando na pasta _root_ do projeto:
``` 
$ scripts/write-sdcard-rpi3.sh <nome_dispositivo>
```

Ao inserir o SD Card na Raspberry e utilizando uma saída para o vídeo, se tudo foi feito corretamente, o Andoid deve bootar normalmente, como pode ser visto abaixo:
<center>![](https://media.discordapp.net/attachments/727592935054639194/786364456643985440/IMG_5310.jpg?width=625&height=469){width=300} ![](https://media.discordapp.net/attachments/727592935054639194/786364450633678888/IMG_5312.jpg?width=625&height=469){width=300} 
![](https://media.discordapp.net/attachments/727592935054639194/786364441004474378/IMG_5313.jpg?width=625&height=469){width=300} </center>


---
