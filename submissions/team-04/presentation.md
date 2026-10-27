# Presentación — team-04 · NET-404 Redes privadas y DNS (≤ 10 min)

## 1. Problema (1 min)

En NexaCloud un backend interno quedó expuesto a Internet: se podía llamar
directamente, sin pasar por la puerta pública.

```
ANTES — el backend tenía puerta a la calle

  Internet ──► nodo:8080 ═══► backend:80
                              (publicado con `ports:`)
```

La causa no fue "la IP del contenedor". Publicar un puerto abre una puerta en
**el nodo**, y Swarm la replica en todos los nodos (routing mesh).

## 2. Arquitectura (2 min)

```
DESPUÉS — solo el frontend tiene puerta

  Internet ──► nodo:18080 ═══► frontend:80      ← única puerta
                                   │
                             private_net (overlay)
                                   │
                               backend:80        ← sin `ports:`
                                   ▲
                             DNS: "backend"
```

- Red `private_net` overlay compartida por ambos servicios.
- `frontend` (whoami, 1 réplica): único servicio con `ports` (`18080:80`).
- `backend` (whoami, 2 réplicas): **sin `ports`**, descubierto por DNS.

El DNS solo traduce nombres; el tráfico va directo a la VIP:

```
frontend ──"¿backend?"──► DNS 127.0.0.11 ──► VIP
     └──────────── HTTP directo ────────────────┘
```

## 3. Demo (3 min)

1. `docker stack deploy -c submissions/team-04/docker-stack.yml team04`
   y `docker stack services team04` → solo `frontend` muestra PORTS.
2. `curl.exe -s http://localhost:18080` → responde el frontend.
3. `docker service inspect team04_backend --format '{{json .Endpoint.Ports}}'`
   → `null`.
4. `docker run --rm --network team04_private_net busybox:1.36 nslookup backend`
   → VIP del servicio (ej. `10.0.2.5`); `tasks.backend` → 2 IPs.
5. `wget http://backend` ×4 → alternan los 2 hostnames (balanceo por VIP).
6. Mismo `wget` fuera de la red → `bad address 'backend'`.

## 4. Decisiones y trade-offs (1.5 min)

- `attachable: true` solo para poder probar; en producción se desactiva.
- Sin `depends_on`: Swarm lo ignora; el orden lo resuelven DNS y reintentos.
- Una sola red: dos redes edge/private no suman seguridad, porque el puerto
  publicado entra por el routing mesh, no por la red del servicio.
- Cambio mínimo: no se agregó proxy ni sidecars; se eliminó una ruta.

## 5. Limitaciones y aprendizajes (1 min)

- Un solo nodo: no se prueba overlay entre hosts ni cifrado.
- Desde el host se llega a la IP interna: no es un puerto publicado, pero
  conviene aclararlo y hacer la prueba negativa desde otra máquina.
- En producción: red `encrypted`, TLS, proxy real hacia `backend`, healthchecks.
- Aprendimos que `ports` expone el servicio en todos los nodos por el routing
  mesh, y que el DNS descubre, no protege.

## 6. Conclusión (30 s)

El incidente no se resolvió ocultando una IP ni agregando un firewall: se
eliminó una ruta que nadie necesitaba. En Swarm, la superficie pública es
exactamente lo que declarás en `ports:`; todo lo demás es interno por defecto.
El DNS no filtra ni redirige: solo pone nombres en el pasillo.
