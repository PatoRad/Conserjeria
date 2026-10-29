# Conserjería · Cerro Paraíso Uno

App web de una sola página (`index.html`) para el registro de visitas, encomiendas y bitácora de la conserjería. Backend en Supabase (Postgres + Auth + RLS).

## Arquitectura

- **Frontend:** `index.html`, sin build. Usa `@supabase/supabase-js@2.58.0` desde jsDelivr.
- **Auth:** correo y contraseña de Supabase Auth. Cada usuario tiene una fila en `perfiles` con su `rol` (`conserje` o `admin`).
- **Turnos:** la sesión de Auth es del equipo de la conserjería. Quién está de turno se define con nombre + PIN (`iniciar_turno`). Los registros se asignan al conserje del turno activo.
- **Seguridad:** la llave publicable está en el cliente, así que toda la protección depende de RLS, permisos por columna, funciones `security definer` y triggers.

## Tablas

| Tabla | Columnas |
|---|---|
| `conserjes` | `id uuid pk`, `nombre text`, `pin_hash text`, `activo bool (true)`, `intentos_fallidos int (0)`, `bloqueado_hasta timestamptz`, `created_at timestamptz` |
| `turnos` | `id uuid pk`, `conserje_id uuid`, `inicio timestamptz (now())`, `fin timestamptz` (nulo = turno abierto) |
| `visitas` | `id`, `nombre`, `depto`, `motivo`, `entrada (now())`, `salida`, `creado_por (auth.uid())`, `anulada (false)`, `motivo_anulacion`, `conserje_id (conserje_de_turno())` |
| `encomiendas` | `id`, `depto`, `remitente`, `nota`, `recibida (now())`, `retirada`, `creado_por`, `anulada`, `motivo_anulacion`, `conserje_id` |
| `bitacora` | `id`, `tipo`, `texto`, `creado (now())`, `creado_por`, `anulada`, `motivo_anulacion`, `conserje_id` |
| `perfiles` | `id uuid pk` (= `auth.users.id`), `nombre text ('Sin nombre')`, `rol text ('conserje')`. Los perfiles los crea la administración; no hay política de INSERT para el cliente. |
| `deptos` | `numero text pk`. Texto, para conservar ceros iniciales (por ejemplo 100, 101… 109). |
| `config` | `id int (1)`, `dias int (90)`. Una sola fila; `dias` es el plazo de retención configurable desde Admin. |
| `auditoria` | `id bigint`, `cuando timestamptz (now())`, `quien uuid (auth.uid())`, `tabla text`, `accion text`, `fila jsonb`. Se llena con el trigger `auditar()`. |

Permisos por columna: el rol `authenticated` solo puede hacer `SELECT` de `conserjes.id`, `nombre` y `activo`. `pin_hash` no es legible desde el cliente. `anon` no tiene permisos sobre `conserjes`.

## Políticas RLS

Helpers: `es_staff()` (conserje o admin), `es_admin()`.

| Tabla | Política | Operación | Condición |
|---|---|---|---|
| visitas, encomiendas, bitacora | `p_v_r`, `p_e_r`, `p_b_r` | SELECT | `es_staff()` |
| visitas, encomiendas, bitacora | `p_v_i`, `p_e_i`, `p_b_i` | INSERT | `es_staff() and creado_por = auth.uid()` |
| visitas, encomiendas, bitacora | `p_v_u`, `p_e_u`, `p_b_u` | UPDATE | `es_staff()` (restringido por trigger, ver abajo) |
| visitas, encomiendas, bitacora | `p_v_d`, `p_e_d`, `p_b_d` | DELETE | `es_admin()` |
| perfiles | `p_perf_r` / `p_perf_w` | SELECT / UPDATE | `es_staff()` / `es_admin()` |
| deptos | `p_dep_r` / `p_dep_w` | SELECT / ALL | `es_staff()` / `es_admin()` |
| config | `p_cfg_r` / `p_cfg_w` | SELECT / UPDATE | `es_staff()` / `es_admin()` |
| auditoria | `p_a_r` | SELECT | `es_admin()` |
| conserjes | `conserjes_leer` | SELECT | `rol_actual() in ('conserje','admin')` |
| turnos | `turnos_leer` | SELECT | `rol_actual() in ('conserje','admin')` |

`conserjes` y `turnos` no tienen políticas de escritura: solo se modifican mediante las funciones.

## Triggers

| Trigger | Tablas | Función | Qué hace |
|---|---|---|---|
| `a0_forzar_insert` (BEFORE INSERT) | visitas, encomiendas, bitacora | `forzar_insert()` | Fuerza `conserje_id` al del turno activo, `anulada = false` y la hora del servidor. Rechaza si no hay turno activo. |
| `a0_proteger_update` (BEFORE UPDATE) | visitas, encomiendas, bitacora | `proteger_update()` | Un conserje solo puede registrar salida/retirada (una vez, con hora del servidor) o anular con motivo. Un registro anulado queda bloqueado. El admin y las sesiones sin usuario (SQL Editor) no tienen restricción. |
| `a1`, `a2`, `a3` | visitas, encomiendas, bitacora | `auditar()` | Guarda cada cambio como JSON en `auditoria`. |

## Registro de quién anula

`visitas`, `encomiendas` y `bitacora` tienen dos columnas más: `anulado_por uuid` (conserje del turno activo al anular; nulo si anuló la administración sin turno) y `anulada_en timestamptz`. Las asigna el servidor en `proteger_update()` cuando `anulada` pasa de falso a verdadero, y `forzar_insert()` las deja nulas al insertar. Las anulaciones anteriores a este cambio no tienen autor. La pestaña Admin las muestra leyendo `auditoria`.

```sql
alter table public.visitas     add column if not exists anulado_por uuid references public.conserjes(id) on delete set null, add column if not exists anulada_en timestamptz;
alter table public.encomiendas add column if not exists anulado_por uuid references public.conserjes(id) on delete set null, add column if not exists anulada_en timestamptz;
alter table public.bitacora    add column if not exists anulado_por uuid references public.conserjes(id) on delete set null, add column if not exists anulada_en timestamptz;
```

Las versiones vigentes de `forzar_insert()` y `proteger_update()` son las del script de "Registro de quién anula (SQL)" al final de este archivo; reemplazan a las del bloque de endurecimiento.

## Funciones

Las funciones de `security definer` fijan `search_path`. `iniciar_turno` usa `crypt()` de la extensión `pgcrypto` (esquema `extensions`).

- `iniciar_turno(p_conserje_id, p_pin)`: valida el PIN, bloquea 5 minutos tras 5 fallos seguidos, cierra el turno abierto anterior y abre uno nuevo. Devuelve el id del turno, o `null` si el PIN es incorrecto.
- `cerrar_turno()`: cierra los turnos abiertos.
- `turno_actual()`: devuelve el turno activo (id, conserje, nombre, inicio). Ignora turnos con más de 14 horas abiertos.
- `conserje_de_turno()`: id del conserje del turno activo. Mismo límite de 14 horas.
- `rol_actual()`, `es_staff()`, `es_admin()`: helpers de rol.
- `auditar()`: trigger de auditoría.

### SQL de las funciones originales

```sql
create or replace function public.auditar() returns trigger
 language plpgsql security definer set search_path to 'public'
as $$ begin
  insert into public.auditoria(tabla,accion,fila)
  values (tg_table_name, tg_op, to_jsonb(coalesce(new,old)));
  return coalesce(new,old);
end $$;

create or replace function public.cerrar_turno() returns void
 language plpgsql security definer set search_path to 'public'
as $$
begin
  if public.rol_actual() not in ('conserje','admin') then
    raise exception 'No autorizado';
  end if;
  update public.turnos set fin = now() where fin is null;
end $$;

create or replace function public.es_admin() returns boolean
 language sql stable security definer set search_path to 'public'
as $$ select exists(select 1 from public.perfiles where id=auth.uid() and rol='admin') $$;

create or replace function public.es_staff() returns boolean
 language sql stable security definer set search_path to 'public'
as $$
  select exists(
    select 1 from public.perfiles
    where id = auth.uid() and rol in ('conserje','admin')
  )
$$;

create or replace function public.rol_actual() returns text
 language sql stable security definer set search_path to 'public'
as $$ select rol::text from public.perfiles where id = auth.uid() $$;

create or replace function public.iniciar_turno(p_conserje_id uuid, p_pin text)
 returns uuid language plpgsql security definer
 set search_path to 'public','extensions'
as $$
declare
  c        public.conserjes;
  v_turno  uuid;
begin
  if public.rol_actual() not in ('conserje','admin') then
    raise exception 'No autorizado';
  end if;

  select * into c from public.conserjes
   where id = p_conserje_id and activo for update;
  if not found then raise exception 'Conserje no válido'; end if;

  if c.bloqueado_hasta is not null and c.bloqueado_hasta > now() then
    raise exception 'Bloqueado temporalmente. Intenta en unos minutos';
  end if;

  if c.pin_hash <> crypt(p_pin, c.pin_hash) then
    -- 5 fallos seguidos => bloqueo de 5 minutos
    update public.conserjes set
      intentos_fallidos = case when intentos_fallidos + 1 >= 5 then 0
                               else intentos_fallidos + 1 end,
      bloqueado_hasta   = case when intentos_fallidos + 1 >= 5
                               then now() + interval '5 minutes'
                               else bloqueado_hasta end
    where id = c.id;
    return null;
  end if;

  update public.conserjes
     set intentos_fallidos = 0, bloqueado_hasta = null
   where id = c.id;

  update public.turnos set fin = now() where fin is null;
  insert into public.turnos (conserje_id) values (c.id)
  returning id into v_turno;
  return v_turno;
end $$;
```

### Endurecimiento (expiración del turno y triggers)

Ejecutar en el SQL Editor de Supabase, después de lo anterior.

```sql
-- 1) Expiración del turno (14 horas)
create or replace function public.conserje_de_turno()
 returns uuid language sql stable security definer set search_path to 'public'
as $$
  select conserje_id from public.turnos
   where fin is null and inicio > now() - interval '14 hours'
   order by inicio desc limit 1
$$;

create or replace function public.turno_actual()
 returns table(turno_id uuid, conserje_id uuid, nombre text, inicio timestamptz)
 language sql stable security definer set search_path to 'public'
as $$
  select t.id, c.id, c.nombre, t.inicio
    from public.turnos t
    join public.conserjes c on c.id = t.conserje_id
   where t.fin is null
     and t.inicio > now() - interval '14 hours'
     and public.rol_actual() in ('conserje','admin')
   order by t.inicio desc
   limit 1
$$;

-- 2) INSERT: el servidor decide conserje, hora y estado
create or replace function public.forzar_insert()
 returns trigger language plpgsql security definer set search_path to 'public'
as $$
begin
  if auth.uid() is null then return new; end if;  -- SQL Editor / mantenimiento

  new.conserje_id := public.conserje_de_turno();
  if new.conserje_id is null then
    raise exception 'No hay un turno activo. Inicia turno para registrar';
  end if;
  new.anulada := false;
  new.motivo_anulacion := null;

  new := jsonb_populate_record(new, jsonb_build_object(
    case tg_table_name when 'visitas' then 'entrada'
                       when 'encomiendas' then 'recibida'
                       else 'creado' end, now()));
  if tg_table_name = 'visitas' then
    new := jsonb_populate_record(new, jsonb_build_object('salida', null));
  elsif tg_table_name = 'encomiendas' then
    new := jsonb_populate_record(new, jsonb_build_object('retirada', null));
  end if;
  return new;
end $$;

-- 3) UPDATE: solo salida/retirada y anulación con motivo
create or replace function public.proteger_update()
 returns trigger language plpgsql security definer set search_path to 'public'
as $$
declare
  v_old jsonb := to_jsonb(old);
  v_new jsonb := to_jsonb(new);
  permitidas text[];
  k text;
begin
  if auth.uid() is null or public.es_admin() then return new; end if;

  permitidas := case tg_table_name
    when 'visitas'     then array['salida','anulada','motivo_anulacion']
    when 'encomiendas' then array['retirada','anulada','motivo_anulacion']
    else array['anulada','motivo_anulacion'] end;

  if (v_new - permitidas) is distinct from (v_old - permitidas) then
    raise exception 'Solo se puede registrar salida/retirada o anular';
  end if;

  if old.anulada then
    raise exception 'Registro anulado: solo administración puede modificarlo';
  end if;
  if new.anulada and coalesce(btrim(new.motivo_anulacion),'') = '' then
    raise exception 'Falta el motivo de la anulación';
  end if;
  if not new.anulada and new.motivo_anulacion is distinct from old.motivo_anulacion then
    raise exception 'No se puede cambiar el motivo sin anular';
  end if;

  foreach k in array array['salida','retirada'] loop
    if v_old ? k then
      if v_old->>k is not null and v_new->k is distinct from v_old->k then
        raise exception 'Ese registro ya tiene % registrada', k;
      elsif v_old->>k is null and v_new->>k is not null then
        new := jsonb_populate_record(new, jsonb_build_object(k, now()));
      end if;
    end if;
  end loop;

  return new;
end $$;

drop trigger if exists a0_forzar_insert on public.visitas;
drop trigger if exists a0_forzar_insert on public.encomiendas;
drop trigger if exists a0_forzar_insert on public.bitacora;
create trigger a0_forzar_insert before insert on public.visitas
  for each row execute function public.forzar_insert();
create trigger a0_forzar_insert before insert on public.encomiendas
  for each row execute function public.forzar_insert();
create trigger a0_forzar_insert before insert on public.bitacora
  for each row execute function public.forzar_insert();

drop trigger if exists a0_proteger_update on public.visitas;
drop trigger if exists a0_proteger_update on public.encomiendas;
drop trigger if exists a0_proteger_update on public.bitacora;
create trigger a0_proteger_update before update on public.visitas
  for each row execute function public.proteger_update();
create trigger a0_proteger_update before update on public.encomiendas
  for each row execute function public.proteger_update();
create trigger a0_proteger_update before update on public.bitacora
  for each row execute function public.proteger_update();
```

Para revertir: `drop trigger a0_forzar_insert / a0_proteger_update` en las tres tablas y volver a crear `conserje_de_turno()` y `turno_actual()` sin el filtro de 14 horas.

## Retención (purga programada)

Requiere la extensión `pg_cron` (Database > Extensions). El job `purga-conserjeria` corre todos los días a las 07:00 UTC y ejecuta `purgar_antiguos()`, que borra:

- Visitas con salida (o anuladas) y encomiendas retiradas (o anuladas) con más de `config.dias` días (mínimo 30).
- Bitácora con más de `config.dias` días.
- Auditoría con más de 365 días.

Las visitas sin salida y las encomiendas sin retirar no se purgan nunca, salvo que estén anuladas. Durante la purga, `auditar()` no registra los borrados (usa `app.purga`), para que no queden copias de lo purgado. La función solo la puede ejecutar el cron, no los usuarios de la app.

```sql
create or replace function public.auditar() returns trigger
 language plpgsql security definer set search_path to 'public'
as $$ begin
  if current_setting('app.purga', true) = '1' then
    return coalesce(new, old);
  end if;
  insert into public.auditoria(tabla, accion, fila)
  values (tg_table_name, tg_op, to_jsonb(coalesce(new, old)));
  return coalesce(new, old);
end $$;

create or replace function public.purgar_antiguos(p_dias_auditoria int default 365)
 returns jsonb language plpgsql security definer set search_path to 'public'
as $$
declare
  d int; v int; e int; b int; a int;
begin
  select greatest(dias, 30) into d from public.config where id = 1;
  d := coalesce(d, 90);

  perform set_config('app.purga', '1', true);

  delete from public.visitas
   where (salida is not null or anulada)
     and entrada < now() - make_interval(days => d);
  get diagnostics v = row_count;

  delete from public.encomiendas
   where (retirada is not null or anulada)
     and recibida < now() - make_interval(days => d);
  get diagnostics e = row_count;

  delete from public.bitacora
   where creado < now() - make_interval(days => d);
  get diagnostics b = row_count;

  perform set_config('app.purga', '0', true);

  delete from public.auditoria
   where cuando < now() - make_interval(days => p_dias_auditoria);
  get diagnostics a = row_count;

  return jsonb_build_object('dias', d, 'visitas', v, 'encomiendas', e,
                            'bitacora', b, 'auditoria', a);
end $$;

revoke execute on function public.purgar_antiguos(int) from public, anon, authenticated;

select cron.schedule('purga-conserjeria', '0 7 * * *',
                     $$ select public.purgar_antiguos() $$);
```

Para revisar las corridas: `select status, return_message, start_time from cron.job_run_details order by start_time desc limit 3;`

## Registro de quién anula (SQL)

```sql
create or replace function public.forzar_insert()
 returns trigger language plpgsql security definer set search_path to 'public'
as $$
begin
  if auth.uid() is null then return new; end if;

  new.conserje_id := public.conserje_de_turno();
  if new.conserje_id is null then
    raise exception 'No hay un turno activo. Inicia turno para registrar';
  end if;
  new.anulada := false;
  new.motivo_anulacion := null;
  new.anulado_por := null;
  new.anulada_en := null;

  new := jsonb_populate_record(new, jsonb_build_object(
    case tg_table_name when 'visitas' then 'entrada'
                       when 'encomiendas' then 'recibida'
                       else 'creado' end, now()));
  if tg_table_name = 'visitas' then
    new := jsonb_populate_record(new, jsonb_build_object('salida', null));
  elsif tg_table_name = 'encomiendas' then
    new := jsonb_populate_record(new, jsonb_build_object('retirada', null));
  end if;
  return new;
end $$;

create or replace function public.proteger_update()
 returns trigger language plpgsql security definer set search_path to 'public'
as $$
declare
  v_old jsonb := to_jsonb(old);
  v_new jsonb := to_jsonb(new);
  permitidas text[];
  k text;
begin
  -- Sello de anulación (también cuando anula el admin)
  if new.anulada and not old.anulada then
    new.anulado_por := public.conserje_de_turno();
    new.anulada_en := now();
  elsif not new.anulada then
    new.anulado_por := null;
    new.anulada_en := null;
  end if;

  if auth.uid() is null or public.es_admin() then return new; end if;

  permitidas := case tg_table_name
    when 'visitas'     then array['salida','anulada','motivo_anulacion']
    when 'encomiendas' then array['retirada','anulada','motivo_anulacion']
    else array['anulada','motivo_anulacion'] end;

  if (v_new - permitidas) is distinct from (v_old - permitidas) then
    raise exception 'Solo se puede registrar salida/retirada o anular';
  end if;

  if old.anulada then
    raise exception 'Registro anulado: solo administración puede modificarlo';
  end if;
  if new.anulada and coalesce(btrim(new.motivo_anulacion),'') = '' then
    raise exception 'Falta el motivo de la anulación';
  end if;
  if not new.anulada and new.motivo_anulacion is distinct from old.motivo_anulacion then
    raise exception 'No se puede cambiar el motivo sin anular';
  end if;

  foreach k in array array['salida','retirada'] loop
    if v_old ? k then
      if v_old->>k is not null and v_new->k is distinct from v_old->k then
        raise exception 'Ese registro ya tiene % registrada', k;
      elsif v_old->>k is null and v_new->>k is not null then
        new := jsonb_populate_record(new, jsonb_build_object(k, now()));
      end if;
    end if;
  end loop;

  return new;
end $$;
```

## Pendiente de documentar

- Número de departamentos y pisos del edificio (los departamentos se cargan desde Admin).

## Verificado

- RLS activado en todas las tablas: `auditoria`, `bitacora`, `config`, `conserjes`, `deptos`, `encomiendas`, `perfiles`, `turnos`, `visitas`.
- `authenticated` solo puede leer `id`, `nombre` y `activo` de `conserjes`.
