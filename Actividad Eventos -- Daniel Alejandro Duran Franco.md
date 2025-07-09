## Actividad -- Daniel Alejandro Duran Franco



Haciendo uso de las siguientes tablas para la base de datos de `pizza` realice los siguientes ejercicios de `Events` centrados en el uso de **ON COMPLETION PRESERVE** y **ON COMPLETION NOT PRESERVE** :

```sql
CREATE TABLE IF NOT EXISTS resumen_ventas (
fecha       DATE      PRIMARY KEY,
total_pedidos INT,
total_ingresos DECIMAL(12,2),
creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id  INT UNSIGNED NOT NULL,
  stock_actual    INT NOT NULL,
  fecha_alerta    DATETIME NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
```

1. Resumen Diario Único : crear un evento que genere un resumen de ventas **una sola vez** al finalizar el día de ayer y luego se elimine automáticamente llamado `ev_resumen_diario_unico`.

   ```sql
   DROP EVENT IF EXISTS ev_resumen_diario_unico;
   
   DELIMITER $$
   CREATE EVENT ev_resumen_diario_unico
   ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 MINUTE
   ON COMPLETION NOT PRESERVE
   DO
   BEGIN
       DECLARE fecha_ayer DATE;
       DECLARE total_pedidos_ayer INT DEFAULT 0;
       DECLARE total_ingresos_ayer DECIMAL(12,2) DEFAULT 0.00;
       
       SET fecha_ayer = DATE_SUB(CURDATE(), INTERVAL 1 DAY);
       
       SELECT 
           COUNT(*) as pedidos,
           COALESCE(SUM(precio_total), 0) as ingresos
       INTO total_pedidos_ayer, total_ingresos_ayer
       FROM pedido 
       WHERE DATE(fecha_pedido) = fecha_ayer;
       
       INSERT IGNORE INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
       VALUES (fecha_ayer, total_pedidos_ayer, total_ingresos_ayer);
       
       INSERT INTO log_eventos (evento, descripcion, fecha_ejecucion) 
       VALUES ('ev_resumen_diario_unico', 
               CONCAT('Resumen generado para ', fecha_ayer, ': ', total_pedidos_ayer, ' pedidos, $', total_ingresos_ayer),
               NOW());
               
   END$$
   DELIMITER ;
   ```

   2. Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, **manteniendo** el evento para que siga ejecutándose cada semana llamado `ev_resumen_semanal`.

      ```sql
      DROP EVENT IF EXISTS ev_resumen_semanal;
      
      DELIMITER $
      CREATE EVENT ev_resumen_semanal
      ON SCHEDULE EVERY 1 WEEK
      STARTS (
          CASE 
              WHEN DAYOFWEEK(CURDATE()) = 2 AND CURTIME() < '01:00:00' THEN 
                  CONCAT(CURDATE(), ' 01:00:00')
              ELSE 
                  CONCAT(DATE_ADD(CURDATE(), INTERVAL (9 - DAYOFWEEK(CURDATE())) DAY), ' 01:00:00')
          END
      )
      ON COMPLETION PRESERVE
      DO
      BEGIN
          DECLARE fecha_inicio_semana DATE;
          DECLARE fecha_fin_semana DATE;
          DECLARE total_pedidos_semana INT DEFAULT 0;
          DECLARE total_ingresos_semana DECIMAL(12,2) DEFAULT 0.00;
          DECLARE semana_anio VARCHAR(20);
          
          SET fecha_fin_semana = DATE_SUB(CURDATE(), INTERVAL DAYOFWEEK(CURDATE()) DAY);  -- Domingo pasado
          SET fecha_inicio_semana = DATE_SUB(fecha_fin_semana, INTERVAL 6 DAY);  -- Lunes pasado
          
          SET semana_anio = CONCAT(YEAR(fecha_inicio_semana), '-S', WEEK(fecha_inicio_semana, 1));
          
          SELECT 
              COUNT(*) as pedidos,
              COALESCE(SUM(precio_total), 0) as ingresos
          INTO total_pedidos_semana, total_ingresos_semana
          FROM pedido 
          WHERE DATE(fecha_pedido) >= fecha_inicio_semana 
          AND DATE(fecha_pedido) <= fecha_fin_semana;
          
          INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos)
          VALUES (fecha_inicio_semana, total_pedidos_semana, total_ingresos_semana)
          ON DUPLICATE KEY UPDATE 
              total_pedidos = VALUES(total_pedidos),
              total_ingresos = VALUES(total_ingresos);
          
          INSERT INTO log_eventos (evento, descripcion, fecha_ejecucion) 
          VALUES ('ev_resumen_semanal', 
                  CONCAT('Resumen semanal generado para ', semana_anio, 
                         ' (', fecha_inicio_semana, ' a ', fecha_fin_semana, '): ', 
                         total_pedidos_semana, ' pedidos, 
      ```

      3. Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (`alerta_stock`) de ingredientes con stock < 5, y luego autodestruir el evento.

         ```sql
         DROP EVENT IF EXISTS ev_alerta_stock_arranque;
         
         DELIMITER $
         CREATE EVENT ev_alerta_stock_arranque
         ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 2 MINUTE
         ON COMPLETION NOT PRESERVE
         DO
         BEGIN
             DECLARE done INT DEFAULT FALSE;
             DECLARE ingrediente_id_actual INT;
             DECLARE stock_actual INT;
             DECLARE ingrediente_nombre VARCHAR(100);
             
             DECLARE cur_stock_bajo CURSOR FOR
                 SELECT id, stock, nombre_ingrediente
                 FROM ingrediente 
                 WHERE stock < 5;
             
             DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
             
             OPEN cur_stock_bajo;
             
             read_loop: LOOP
                 FETCH cur_stock_bajo INTO ingrediente_id_actual, stock_actual, ingrediente_nombre;
                 
                 IF done THEN
                     LEAVE read_loop;
                 END IF;
                 
                 INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
                 VALUES (ingrediente_id_actual, stock_actual, NOW());
                 
             END LOOP;
             
             CLOSE cur_stock_bajo;
             
             INSERT INTO log_eventos (evento, descripcion, fecha_ejecucion) 
             VALUES ('ev_alerta_stock_arranque', 
                     CONCAT('Verificación de stock completada - Alertas generadas para ingredientes con stock < 5'),
                     NOW());
                     
         END$
         DELIMITER ;
         
         ```

         4. Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en `alerta_stock`, **dejando** el evento activo para siempre llamado `ev_monitor_stock_bajo`.

            ```sql
            DROP EVENT IF EXISTS ev_monitor_stock_bajo;
            
            DELIMITER $
            CREATE EVENT ev_monitor_stock_bajo
            ON SCHEDULE EVERY 30 MINUTE
            STARTS CURRENT_TIMESTAMP + INTERVAL 1 MINUTE
            ON COMPLETION PRESERVE
            DO
            BEGIN
                INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta)
                SELECT 
                    id,
                    stock,
                    NOW()
                FROM ingrediente 
                WHERE stock < 10;
                
            END$
            DELIMITER ;
            ```

            5. Limpieza de Resúmenes Antiguos: una sola vez, eliminar de `resumen_ventas` los registros con fecha anterior a hace 365 días y luego borrar el evento llamado `ev_purgar_resumen_antiguo`.

               ```sql
               DELIMITER $$
               
               CREATE EVENT IF NOT EXISTS ev_purgar_resumen_antiguo
               ON SCHEDULE AT NOW() + INTERVAL 5 SECOND
               ON COMPLETION NOT PRESERVE
               DO
               BEGIN
                   DELETE FROM resumen_ventas 
                   WHERE fecha < DATE_SUB(CURDATE(), INTERVAL 365 DAY);
                   
                   INSERT INTO log_eventos (evento, descripcion, fecha_ejecucion) 
                   VALUES ('ev_purgar_resumen_antiguo', 
                           CONCAT('Eliminados registros anteriores a ', DATE_SUB(CURDATE(), INTERVAL 365 DAY)), 
                           NOW());
               END$$
               
               DELIMITER ;
               ```

               

​	



