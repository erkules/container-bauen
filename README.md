# Ich

* erkan@linsenraum.de
* xing/linkedin
* http://linsenraum.de 

Und ja:  Freiberufler


# Ziel

* Docker/LXC/systemd-nspawn/Rkt
* Alles nur Linux
* Jeder gute Admin sollte eh alles kennen was jetzt kommt.

# Container vs. VM

* VMs eumulieren Hardware 
* Ein Container ist nur ein Prozess auf dem Host
* Was immer das auch heißt


# Docker

Was haben wir?

Ist doch wie eine VM!

~~~
docker container run -ti --name vorlage alpine
~~~

* Prozessraum
* Prozesse
* Filesystem

# Filesystem? chroot!

~~~
mkdir /tmp/container
docker container export vorlage | tar xf - -C /tmp/container
~~~

Warum wollte ich?:

~~~
chroot /tmp/container zsh
~~~

Problem mit chroot?

Wir haben quasi keine Isolierung `¯\_(ツ)_/¯`

# Namespaces

Genereller Überblick:

~~~
readlink /proc/self/ns/*
~~~

Schauen wir uns das erst mal ohne 
chroot an. 
Aber am Ende:

Imho noch ein `pstree` mitlaufen lassen: 

~~~
unshare -p -f -m -u -n  -i
chroot /tmp/container sh 
ps ax .. und ein Hups
~~~

Genau das machen was da steht. (`mount -t proc proc /proc`)

Und schauen ob andere Prozess was in /tmp/container/proc sehen :)

# Mount

Wo wir schon bei Mount sind.

~~~
docker container run --volume /var/tmp:/srv  --rm -ti ubuntu
~~~

* Oben beim /proc haben wir ja schon einen mount gesehen.
* Hier ist es genau so nur *vor* dem chroot
* Und ja nach dem unshare damit sonst niemand den mount mitbekommt \o/

~~~
mkdir /tmp/container/haha
mount  --bind /var/tmp /tmp/container/haha
unshare -p -f -m -u -n  -i
chroot /tmp/container sh 
ls /haha
~~~

# Cgroups

Am Beispiel pids (limit)

~~~
docker container run --rm -ti --pids-limit 5  ubuntu
~~~

Achso: wieder ein `pstree` mitlaufen lassen :)

## Cgroups v1

Sollte nicht mehr funzeen

~~~
mkdir /sys/fs/cgroup/pids/lala
echo 5 >/sys/fs/cgroup/pids/lala/pids.max

echo $$ >/sys/fs/cgroup/pids/lala/tasks
besser? 
echo $ContainerPid  >/sys/fs/cgroup/pids/lala/tasks
~~~

## Cgroups v2

~~~
mkdir /sys/fs/cgroup/container
echo $ContainerPid > /sys/fs/cgroup/container/cgroup.procs
echo 5 >/sys/fs/cgroup/container/pids.max
~~~

Ruhig ein paar Prozesse im Container starten. Klappt schnell nicht 😎

# Artefakt

Aber jeder Container schreibt doch ein einen eigenen Layer

~~~
docker container run --rm -d --name www erkules/nginxhostname
docker container ps -s -f name=www
~~~

Gleich 2x

~~~
mkdir /tmp/upper1 /tmp/work1 /tmp/runningcontainer1

mount -t overlay overlay -o lowerdir=/tmp/container,upperdir=/tmp/upper1,workdir=/tmp/work1  /tmp/runningcontainer1 
~~~
~~~
mkdir /tmp/upper2 /tmp/work2 /tmp/runningcontainer2
mount -t overlay overlay -o lowerdir=/tmp/container,upperdir=/tmp/upper2,workdir=/tmp/work2  /tmp/runningcontainer2 
~~~



# Netzwerk?

FYI: Das ist jetzt total abgedreht!

FYI: `ip netns` macht auch nur `unshare` `¯\_(ツ)_/¯`

Vielleicht statt <<ContainerPID>> irgend ein `unshare -n`?

Oder wir haben pstree noch am Laufen ...



~~~
mkdir /var/run/netns <- Falls nicht vorhanden. Braucht iproute2
ln -s /proc/<<ContainerPID>>/ns/net /var/run/netns/hallo
ip netns exec hallo ip a s
ip link add hostende type veth peer name containerende
ip link set containerende  netns hallo
ip netns exec hallo ip a s
ip netns exec hallo ip link set containerende up
ip netns exec hallo ip a add 172.40.1.1/26 dev containerende
                    ip link set hostende up
                    ip a add 172.40.1.2/26 dev hostende
~~~

Pingt ...

aber noch lange nicht fertig ... ;)


# nsenter

Haben wir noch Zeit?

Und wenn schon!

* Container ohne Network starten. Mit nsenter Paketer installieren
* Netzwerk in einem Container mit Host-Tools debuggen


# Conclusio

* Wir haben die ganz Zeit *einen* Prozess geändert
* Von wegen VM :)
* I.e. Security ist 0815 Linux-Admin-Knowhow

# Links

[bocker](https://github.com/p8952/bocker)
