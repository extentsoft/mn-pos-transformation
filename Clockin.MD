

CREATE COMPUTE MODULE SA_Clockin_FormParam
	CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
		-- CALL CopyMessageHeaders();
		-- CALL CopyEntireMessage();
		-- PROPAGATE;
		
		
		IF CAST(InputRoot.JSON.Data.meta.total_count AS INTEGER) > 0 THEN
			DECLARE count INTEGER;
			DECLARE worklog_count INTEGER;
			DECLARE i INTEGER 1;
			SET count = CARDINALITY(InputRoot.JSON.Data.objects.Item[]);
			SET count = count + 1;
			
			DECLARE my_credentials ROW;
			DECLARE worklog ROW;
			DECLARE employee_info ROW;
			SET my_credentials.Item[] = (SELECT CC.SYSTEM_KEY AS SYSTEM_KEY FROM Database.MINORDB.db2inst1.C_CREDENTIALS AS CC, Database.MINORDB.db2inst1.C_ENVIRONMENT AS CE WHERE CC.SYSTEM_CODE = CE.REFCODE AND CE.SYSTEM_NAME = 'Tanda' AND CE.FLAG = 1 );	
			SET OutputLocalEnvironment.Destination.REST.Request.Parameters."Authorization" = my_credentials.Item[1].SYSTEM_KEY;			
			
			DECLARE e_row ROW;
			DECLARE e_row_count, e_emp_count INTEGER;	
			DECLARE Tpattern CHARACTER 'yyyy-MM-dd HH:mm:ss';
			DECLARE EPOCH_TIME TIMESTAMP;
			SET EPOCH_TIME = CAST('1970-01-01T00:00:00.000+00.00' AS TIMESTAMP FORMAT 'IU');
			
			WHILE i < count DO
				-- check if records exists 
				-- SET OutputRoot.JSON.Data.key = my_credentials.Item[1].SYSTEM_KEY;	
				-- PROPAGATE TO TERMINAL 'out1';
				
				
				SET OutputLocalEnvironment.Destination.REST.Request.Parameters."Authorization" = my_credentials.Item[1].SYSTEM_KEY;	
				SET employee_info.Item[] = (SELECT ME.ID AS EID FROM Database.MINORDB.db2inst1.M_EMPLOYEE AS ME WHERE ME.RESOURCE_URI = InputRoot.JSON.Data.objects.Item[i].employee );		
				
				
				SET e_emp_count = CARDINALITY(employee_info.Item[]);
				SET e_row.Item[] = (
					SELECT MW.CREATED_DATE AS CREATED_DATE 
					FROM Database.MINORDB.db2inst1.M_WORKLOG AS MW 
					WHERE 
						MW.CREATED_DATE = REPLACE(InputRoot.JSON.Data.objects.Item[i].created_date, 'T', ' ') 
						AND MW.CLOCK_IN = REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_in, 'T', ' ') 
						AND MW.CLOCK_OUT = REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_out, 'T', ' ') 
					);
					
				SET e_row_count = CARDINALITY(e_row.Item[]);
				
				-- IGNORE RECORDS PROCESSED 
				IF e_row_count = 0 OR e_emp_count <= 0 THEN
					
					IF InputRoot.JSON.Data.objects.Item[i].clock_out IS NULL THEN
						-- *** PERFORM CLOCKIN *** 
					
					
						DECLARE MyTimeStamp TIMESTAMP;
						DECLARE SourceTimeStamp CHARACTER;
						SET SourceTimeStamp =  REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_in, 'T', ' ');
						SET MyTimeStamp = CAST (SourceTimeStamp AS TIMESTAMP FORMAT Tpattern);
						
						SET OutputRoot.JSON.Data.user_id = employee_info.Item[1].EID;
						SET OutputRoot.JSON.Data.type = 'clockin';
						SET OutputRoot.JSON.Data.time = CAST( (MyTimeStamp - EPOCH_TIME)SECOND AS INTEGER);
		
						
						-- EXAMPLE
						-- DECLARE iso_pattern CHARACTER 'yyyy-MM-dd HH:mm:ss';
						-- DECLARE d1 CHARACTER '2018-08-19T13:50:22';
						-- SET d1 = REPLACE(d1,'T',' ');
						-- DECLARE t1 TIMESTAMP;
						-- SET t1 = CAST(d1 AS TIMESTAMP FORMAT iso_pattern);	
						-- DECLARE EPOCH_TIME, IN_TIMESTAMP TIMESTAMP;
						-- SET EPOCH_TIME = CAST('1970-01-01T00:00:00.000+07.00' AS TIMESTAMP FORMAT 'IU');
						-- SET IN_TIMESTAMP = t1;
						-- SET OutputRoot.JSON.Data.i1 = CAST( (IN_TIMESTAMP - EPOCH_TIME)SECOND AS INTEGER);
						
						
						INSERT INTO Database.MINORDB.db2inst1.M_WORKLOG(clock_in, created_date, updated_date, resource_uri_employee) 
						VALUES(
							REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_in, 'T', ' '), 
							REPLACE(InputRoot.JSON.Data.objects.Item[i].created_date, 'T', ' '),
							REPLACE(InputRoot.JSON.Data.objects.Item[i].updated_date, 'T', ' '), 
							InputRoot.JSON.Data.objects.Item[i].employee);
						COMMIT;
					
						-- INSERT INTO Database.MINORDB.db2inst1.M_WORKLOG(clock_in, created_date, resource_uri_employee) 
						-- VALUES(InputRoot.JSON.Data.objects.Item[i].clock_in, InputRoot.JSON.Data.objects.Item[i].created_date, InputRoot.JSON.Data.objects.Item[i].employee);
						-- COMMIT;
						
						
						PROPAGATE TO TERMINAL 'out' ENVIRONMENT OutputLocalEnvironment MESSAGE OutputRoot;
					ELSE 
					
						-- 1. check if clock_in is existed					
						DECLARE source CHARACTER;
						SET source = REPLACE(InputRoot.JSON.Data.objects.Item[i].created_date, 'T', ' ');
						SET worklog.Item[] = (SELECT MW.clock_in FROM Database.MINORDB.db2inst1.M_WORKLOG AS MW WHERE MW.created_date = source );
						SET worklog_count = CARDINALITY(worklog.Item[]);
						
						
						
						IF worklog_count > 0 THEN
							-- *** PERFORM ONLY CLOCKOUT ***
							
							DECLARE MyTimeStamp TIMESTAMP;
							DECLARE SourceTimeStamp CHARACTER;
							SET SourceTimeStamp =  REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_out, 'T', ' ');
							SET MyTimeStamp = CAST (SourceTimeStamp AS TIMESTAMP FORMAT Tpattern);
							SET OutputRoot.JSON.Data.user_id = employee_info.Item[1].EID;
							SET OutputRoot.JSON.Data.type = 'clockout';
							SET OutputRoot.JSON.Data.time = CAST( (MyTimeStamp - EPOCH_TIME)SECOND AS INTEGER);
							
							
							
							UPDATE Database.MINORDB.db2inst1.M_WORKLOG AS MW 
							SET 
								CLOCK_OUT = REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_out, 'T', ' '), 
								UPDATED_DATE = REPLACE(InputRoot.JSON.Data.objects.Item[i].updated_date, 'T', ' ')
							WHERE MW.CREATED_DATE = REPLACE(InputRoot.JSON.Data.objects.Item[i].created_date, 'T', ' ');
							
							COMMIT;
							
							
							PROPAGATE TO TERMINAL 'out' ENVIRONMENT OutputLocalEnvironment MESSAGE OutputRoot;
						ELSE
							-- *** PERFORM BOTH CLOCKIN and CLOCKOUT *** 
							DECLARE MyTimeStamp TIMESTAMP;
							DECLARE SourceTimeStamp CHARACTER;
							
							SET SourceTimeStamp =  REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_in, 'T', ' ');
							SET MyTimeStamp = CAST (SourceTimeStamp AS TIMESTAMP FORMAT Tpattern);
							SET OutputRoot.JSON.Data.user_id = employee_info.Item[1].EID;
							SET OutputRoot.JSON.Data.type = 'clockin';
							SET OutputRoot.JSON.Data.time = CAST( (MyTimeStamp - EPOCH_TIME)SECOND AS INTEGER);
							PROPAGATE TO TERMINAL 'out' ENVIRONMENT OutputLocalEnvironment MESSAGE OutputRoot;
							
							SET SourceTimeStamp =  REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_out, 'T', ' ');
							SET MyTimeStamp = CAST (SourceTimeStamp AS TIMESTAMP FORMAT Tpattern);
							SET OutputRoot.JSON.Data.user_id = employee_info.Item[1].EID;
							SET OutputRoot.JSON.Data.type = 'clockout';
							SET OutputRoot.JSON.Data.time = CAST( (MyTimeStamp - EPOCH_TIME)SECOND AS INTEGER);
							PROPAGATE TO TERMINAL 'out' ENVIRONMENT OutputLocalEnvironment MESSAGE OutputRoot;
							
							
							INSERT INTO Database.MINORDB.db2inst1.M_WORKLOG(clock_in, clock_out, created_date, updated_date, resource_uri_employee) 
							VALUES(
								REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_in, 'T', ' '), 
								REPLACE(InputRoot.JSON.Data.objects.Item[i].clock_out, 'T', ' '), 
								REPLACE(InputRoot.JSON.Data.objects.Item[i].created_date, 'T', ' '),
								REPLACE(InputRoot.JSON.Data.objects.Item[i].updated_date, 'T', ' '), 
								InputRoot.JSON.Data.objects.Item[i].employee);
							
							COMMIT;
						
						END IF;
					
					END IF;
			
				END IF;
				
					
				
				SET i = i + 1;
			END WHILE;
		ELSE
			CALL CopyEntireMessage();
			PROPAGATE;
		END IF;
		
		RETURN FALSE;
	END;

	CREATE PROCEDURE CopyMessageHeaders() BEGIN
		DECLARE I INTEGER 1;
		DECLARE J INTEGER;
		SET J = CARDINALITY(InputRoot.*[]);
		WHILE I < J DO
			SET OutputRoot.*[I] = InputRoot.*[I];
			SET I = I + 1;
		END WHILE;
	END;

	CREATE PROCEDURE CopyEntireMessage() BEGIN
		SET OutputRoot = InputRoot;
	END;
END MODULE;
