# MikroTik – DHCP z rezerwacją IP po MAC + Internet na ether5

## Cel

Komputer ma pozostać ustawiony na DHCP, ale MikroTik ma zawsze przydzielać mu ten sam adres IP na podstawie adresu MAC. Dodatkowo komputer ma mieć dostęp do Internetu przez MikroTika.

## Założenia

```text
Internet / sieć nadrzędna
192.168.10.0/24
        |
      WAN1
   DHCP Client
        |
     MikroTik
        |
      ether5
192.168.20.254/24
        |
       PC
DHCP: 192.168.20.22
MAC: AC:1A:3D:B7:13:35
```

> Ważne: sieć WAN i LAN MikroTika muszą być różne.  
> WAN pracuje w `192.168.10.0/24`, a LAN na `ether5` w `192.168.20.0/24`.

## 1. WAN – dostęp MikroTika do Internetu

Port `WAN1` powinien być niezależnym interfejsem i nie należeć do bridge'a LAN.

W **Bridge → Ports** usuń `WAN1` z bridge'a, jeżeli się tam znajduje.

Następnie:

**IP → DHCP Client → +**

Ustaw:

```text
Interface: WAN1
Use Peer DNS: Yes
Add Default Route: Yes
Default Route Distance: 1
```

Po podłączeniu przewodu z sieci nadrzędnej status powinien zmienić się na:

```text
bound
```

Przykładowo MikroTik może otrzymać:

```text
IP WAN: 192.168.10.x
Gateway: 192.168.10.254
```

Sprawdź dostęp do Internetu z MikroTika:

**Tools → Ping**

```text
1.1.1.1
```

## 2. Adres LAN na ether5

Wejdź w:

**IP → Addresses → +**

Ustaw:

```text
Address: 192.168.20.254/24
Interface: ether5
```

MikroTik utworzy sieć:

```text
192.168.20.0/24
```

Adres `192.168.20.254` będzie bramą dla urządzeń podłączonych do `ether5`.

## 3. Serwer DHCP na ether5

Wejdź w:

**IP → DHCP Server → DHCP Setup**

Wybierz:

```text
Interface: ether5
```

Ustaw:

```text
DHCP Network: 192.168.20.0/24
Gateway: 192.168.20.254
Addresses to Give Out: 192.168.20.100-192.168.20.200
DNS: 1.1.1.1, 8.8.8.8
Lease Time: 1d
```

Adres `192.168.20.22` pozostaje poza pulą dynamiczną, ponieważ będzie zarezerwowany dla konkretnego PC.

## 4. Rezerwacja DHCP po MAC

Podłącz PC do `ether5`.

Na komputerze pozostaw:

```text
Uzyskaj adres IP automatycznie
Uzyskaj adres serwera DNS automatycznie
```

W MikroTiku przejdź do:

**IP → DHCP Server → Leases**

Po pojawieniu się dynamicznej dzierżawy dla PC znajdź wpis z MAC:

```text
AC:1A:3D:B7:13:35
```

Wybierz:

**Make Static**

Następnie ustaw:

```text
Address: 192.168.20.22
MAC Address: AC:1A:3D:B7:13:35
Server: dhcp1
```

Efekt:

```text
AC:1A:3D:B7:13:35 → 192.168.20.22
```

PC nadal korzysta z DHCP, ale zawsze otrzymuje ten sam adres.

## 5. NAT – Internet dla sieci ether5

Wejdź w:

**IP → Firewall → NAT → +**

Zakładka **General**:

```text
Chain: srcnat
Src. Address: 192.168.20.0/24
Out. Interface: WAN1
```

Zakładka **Action**:

```text
Action: masquerade
```

Dzięki temu urządzenia z sieci `192.168.20.0/24` mogą wychodzić do Internetu przez `WAN1`.

## 6. Test na komputerze

Na czas testu wyłącz Wi-Fi, aby komputer korzystał wyłącznie z kabla.

W CMD:

```cmd
ipconfig /release
ipconfig /renew
ipconfig
```

Oczekiwany wynik:

```text
IPv4 Address:    192.168.20.22
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.20.254
```

Test komunikacji z MikroTikiem:

```cmd
ping 192.168.20.254
```

Test Internetu:

```cmd
ping 1.1.1.1
```

Test DNS:

```cmd
ping google.com
```

Jeżeli wszystkie trzy testy działają, konfiguracja jest poprawna.

## Efekt końcowy

- PC pozostaje skonfigurowany przez DHCP.
- MikroTik zawsze przydziela PC adres `192.168.20.22`.
- Rezerwacja działa po MAC `AC:1A:3D:B7:13:35`.
- Inne urządzenia mogą otrzymywać adresy z puli `192.168.20.100-192.168.20.200`.
- Dostęp do Internetu realizowany jest przez NAT na `WAN1`.

## Najważniejsza zasada

Nie używaj tej samej podsieci po stronie WAN i LAN MikroTika.

W tym przykładzie:

```text
WAN: 192.168.10.0/24
LAN: 192.168.20.0/24
```
