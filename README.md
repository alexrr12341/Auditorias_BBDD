# Auditorias_BBDD

Práctica individual

Puntuación máxima: 75 puntos

Realiza y documenta adecuadamente las siguientes operaciones:

1. Activa desde SQL*Plus la auditoría de los intentos de acceso fallidos al sistema. Comprueba su funcionamiento.

Por defecto Oracle tiene activada la auditoría,podemos comprobarlo con el siguiente comando:

```sql
SQL> select name , value from v$parameter where name like 'audit_trail';

NAME										 VALUE
-------------------------------------------------------------------------------- --------------------
audit_trail									 DB
```

En caso de que estuviera apagado, podriamos activarlo con el siguiente comando
```sql
ALTER SYSTEM SET audit_trail=db scope=spfile;
```

Para comprobar el funcionamiento vamos a activar la auditoria que compruebe los accesos al sistema

```sql
SQL> AUDIT CREATE SESSION WHENEVER NOT SUCCESSFUL;

Auditoría terminada correctamente.

```

Vamos a comprobar su funcionamiento:

```sql
SQL> connect system/asdfiansdfiao
ERROR:
ORA-01017: nombre de usuario/contraseña no válidos; conexión denegada


SQL> SELECT os_username,username,extended_timestamp,action_name,returncode
  2  FROM dba_audit_session;

OS_USERNAME	USERNAME   EXTENDED_TIMESTAMP							       ACTION_NAME		    RETURNCODE
--------------- ---------- --------------------------------------------------------------------------- ---------------------------- ----------
oracle		SYSTEM	   31/01/20 14:58:33,429817 +01:00					       LOGON			    ##########

```


2. Realiza un procedimiento en PL/SQL que te muestre los accesos fallidos junto con el motivo de los mismos, transformando el código de error almacenado en un mensaje de texto comprensible.

SELECT username, returncode, timestamp
    FROM dba_audit_session 
    WHERE action_name='LOGON' 
    AND returncode != 0 ;

```sql

Create or replace function DarCodigoFallido(p_codigo number)
return varchar2
is
begin
	case p_codigo
	When 1017 then
		return 'Contraseña Equivocada';
	When 28000 then
		return 'Cuenta bloqueada';
	else
		return 'Error desconocido';
	end case;
end;
/

Create or replace procedure MostrarAccesosFallidos
is
	cursor c_codigos is
	Select username, returncode, timestamp
   	From dba_audit_session 
 	Where action_name='LOGON' 
 	And returncode != 0 ;
	v_acceso varchar2(40);
begin
	dbms_output.put_line('Auditorias');
	dbms_output.put_line('----------');
	dbms_output.put_line('Usuario'||chr(9)||'Motivo'||chr(9)||'Fecha');
	dbms_output.put_line('-------'||chr(9)||'------'||chr(9)||'-----');
	for v_codigos in c_codigos loop
		v_acceso:=DarCodigoFallido(v_codigos.returncode);
		dbms_output.put_line(v_codigos.username||chr(9)||v_acceso||chr(9)||to_char(v_codigos.timestamp,'YYYY/MM/DD HH24:MI'));
	end loop;
end;
/
```

3. Activa la auditoría de las operaciones DML realizadas por SCOTT. Comprueba su funcionamiento.

Vamos a activar las auditorias de insert,update y delete de scott.

```sql
Audit insert table, update table, delete table by SCOTT;
```

Vamos a entrar a Scott y vamos a insertar datos en la tabla emp, updatear dicho dato y borrar dicho dato.

```sql
SQL> connect SCOTT/TIGER
Conectado.

Sesión modificada.


SQL> insert into EMP(empno) values ('1234');

1 fila creada.

SQL> update Emp
  2  set empno='123'
  3  where empno='1234';

1 fila actualizada.

SQL> delete from emp where empno='123';

1 fila suprimida.

```
Vamos a observar las auditorias realizadas

```sql
select OS_USERNAME, username, action_name, timestamp
from dba_audit_object
where username='SCOTT';

OS_USERNAME	USERNAME   ACTION_NAME			TIMESTAM
--------------- ---------- ---------------------------- --------
oracle		SCOTT	   INSERT			31/01/20
oracle		SCOTT	   UPDATE			31/01/20
oracle		SCOTT	   DELETE			31/01/20

```


4. Realiza una auditoría de grano fino para almacenar información sobre la inserción de empleados del departamento 10 en la tabla emp de scott.

Para crear una auditoria de grano fino vamos a crear un procedimiento que ejecute dicha creación

```sql
Create or replace procedure GranoFinoEMP
is
begin
DBMS_FGA.ADD_POLICY (
	object_schema      =>  'SCOTT',
	object_name        =>  'EMP',
	policy_name        =>  'InsertenEmp',
	audit_condition    =>  'DEPTNO = 10',
	statement_types    =>  'INSERT');
end;
/
```

```sql
SQL> exec GranoFinoEMP

Procedimiento PL/SQL terminado correctamente.

```

Vamos a insertar una fila en EMP para comprobar su funcionamiento
```sql
SQL> INSERT INTO EMP VALUES(1234, 'Alejandro', 'SYSADMIN', 4543,TO_DATE('02-ABR-1998','DD-MON-YYYY'), 87600, NULL, 10);

1 fila creada.

```

Vamos a comprobar que se ha realizado dicha auditoría

```sql
Select sql_text
From dba_fga_audit_trail
Where policy_name='INSERTENEMP';

SQL_TEXT
--------------------------------------------------
INSERT INTO EMP VALUES(1234, 'Alejandro', 'SYSADMI
N', 4543,TO_DATE('02-ABR-1998','DD-MON-YYYY'), 876
00, NULL, 10)

```

5. Explica la diferencia entre auditar una operación by access o by session.

By access->La auditoría nos almacenará todas las acciones realizadas, sean repetidas o no.
By session->Nos almacenará solo una acción si repetimos la misma opción muchas veces.

6. Documenta las diferencias entre los valores db y db, extended del parámetro audit_trail de ORACLE. Demuéstralas poniendo un ejemplo de la información sobre una operación concreta recopilada con cada uno de ellos.
Según la documentación de Oracle, las diferencias son las siguientes:

-db : Dirige los registros de auditoría a la pista de auditoría de la base de datos (la tabla SYS.AUD $), excepto los registros que siempre se escriben en la pista de auditoría del sistema operativo.

-db, extended: Realiza todas las acciones de la anterior, y también rellena las columnas de SQLBIND y SQLTEXT CLOB de la tabla SYS.AUD $, cuando están disponibles. Estas dos columnas se rellenan solo cuando se especifica este parámetro.

Si queremos activar la opción de db, extended, simplemente tendremos que hacer:

```sql
ALTER SYSTEM SET audit_trail = "DB_EXTENDED" SCOPE=SPFILE
```
Y reiniciar la base de datos

```sql
shutdown
startup
```

7. Localiza en Enterprise Manager las posibilidades para realizar una auditoría e intenta repetir con dicha herramienta los apartados 1, 3 y 4.
8. Averigua si en Postgres se pueden realizar los apartados 1, 3 y 4. Si es así, documenta el proceso adecuadamente.

Para habilitar las auditorias en postgresql primero debemos descargarnos un programa llamado Audit trigger 91plus, para ello

```
postgres@pc-alex:~$ wget wget https://raw.githubusercontent.com/2ndQuadrant/audit-trigger/master/audit.sql

```

Entramos a psql y ejecutamos la siguiente acción
```
postgres=# \i audit.sql
```

Y ejecutamos el siguiente script de creacion de tabla
```
Create table Viveros
(
Codigo Varchar(5),
Direccion Varchar(150),
Telefono Varchar(9),
CONSTRAINT pk_Codigo_Vivero PRIMARY KEY (Codigo),
CONSTRAINT direccionvivero_correcta check(Direccion like '%(Dos Hermanas)' or Direccion like '%(Alcala)' or Direccion like '%(Gelves)')
);
```

Ponemos la recogida de datos de la tabla Viveros
```
SELECT audit.audit_table('Viveros');
```

Y miramos las auditorias:

```
postgres=# insert into Viveros(Codigo,Direccion,Telefono)
values('11127','ME encata jugar al LoL2(Gelves)','955600976');
INSERT 0 1
postgres=# select session_user_name, action, table_name, action_tstamp_clk, client_query
from audit.logged_actions;
 session_user_name | action | table_name |       action_tstamp_clk       |                          client_query                          
-------------------+--------+------------+-------------------------------+----------------------------------------------------------------
 postgres          | I      | viveros    | 2020-02-25 13:19:22.972733+01 | insert into Viveros(Codigo,Direccion,Telefono)                +
                   |        |            |                               | values('11127','ME encata jugar al LoL2(Gelves)','955600976');
(1 fila)
```

9. Averigua si en MySQL se pueden realizar los apartados 1, 3 y 4. Si es así, documenta el proceso adecuadamente.

En mariadb se pueden auditar las acciones dml mediante triggers por lo que vamos a crear una base de datos para las auditorias.

```sql
MariaDB [(none)]> create database viveros;
Query OK, 1 row affected (0.000 sec)

MariaDB [viveros]> Create table Viveros
    -> (
    -> Codigo Varbinary(10),
    -> Direccion Varbinary(150),
    -> Telefono Varbinary(9),
    -> CONSTRAINT pk_Codigo_Vivero PRIMARY KEY (Codigo),
    -> CONSTRAINT direccionvivero_correcta check(Direccion like '%(Dos Hermanas)%' or Direccion like '%(Alcala)%' or Direccion like '%(Gelves)%')
    -> );
Query OK, 0 rows affected, 1 warning (0.565 sec)
```

Ahora permitimos las auditorias:
```sql
SET SESSION profiling = 1;
```

Insertamos datos y miramos si funciona:
```sql
MariaDB [viveros]> insert into Viveros(Codigo) values ('1234');
Query OK, 1 row affected (0.053 sec)

MariaDB [viveros]> show profiles;
+----------+------------+---------------------------------------------------------------------------+
| Query_ID | Duration   | Query                                                                     |
+----------+------------+---------------------------------------------------------------------------+
|        1 | 0.00010116 | insert into Viveros('3241','Hola','32342332')                             |
|        2 | 0.00024342 | insert into Viveros values('3241','Hola','32342332')                      |
|        3 | 0.00023753 | insert into Viveros values('3241','Hola Dos Hermanas','32342332')         |
|        4 | 0.00023927 | insert into Viveros values('3241','Hola Dos Hermanas,Sevilla','32342332') |
|        5 | 0.00022087 | insert into Viveros values('3241','Alcala','32342332')                    |
|        6 | 0.05315302 | insert into Viveros(Codigo) values ('1234')                               |
+----------+------------+---------------------------------------------------------------------------+
6 rows in set (0.000 sec)


```

Para instalar realizar el ejercicio 1 y 3 debemos realizar lo siguiente
```sql
INSTALL SONAME 'server_audit';
SET GLOBAL server_audit_events = 'CONNECT,QUERY,TABLE';
```

10.  Averigua las posibilidades que ofrece MongoDB para auditar los cambios que va sufriendo un documento.
11.  Averigua si en MongoDB se pueden auditar los accesos al sistema.
