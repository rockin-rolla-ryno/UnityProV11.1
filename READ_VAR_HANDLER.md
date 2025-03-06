```text
READ_VAR_HANDLER function in Unity Pro v11.1 its function to manage READ_VAR transactions for communication handling
🔹INPUT PINS 
- COMMS_MANAGEMENT[] → ARRAY OF INT (Holds communication status and data)
- ACTIVITY_BIT_MASK → INT (Mask for checking active communication)
- CANCEL_BIT_MASK → INT (Mask to cancel the READ_VAR transaction)
- STUCK_RESET_BIT_MASK → INT (Mask for toggling reset when communication is stuck)
- MAX_RETRY → INT (Defines the maximum number of retries)
- TIMEOUT_LIMIT → INT (Defines how long to wait before canceling the transaction)
🔹 OUTPUT PINS
- LOG_MESSAGE → STRING (Stores status/error messages)
- ERROR_FLAG → BOOL (Indicates if an error is active)
- TIMEOUT_COUNTER → INT (Tracks timeout attempts)
- RETRY_COUNT → INT (Tracks the number of retry attempts)
🔹 INTERNAL/PRIVATE Variables 
- CommunicationReport → INT (Extracted from COMMS_MANAGEMENT[1])
- OperationReport → INT (Extracted from COMMS_MANAGEMENT[1])
- Prev_CommsState → INT (Stores the last state of COMMS_MANAGEMENT[0])
- Prev_CommReport → INT (Stores the last known CommunicationReport)
- Prev_CommStatus → INT (Stores the last known COMMS_MANAGEMENT[2])
- Prev_CommData → INT (Stores the last known COMMS_MANAGEMENT[3])
- BitStuckTimer → TIMER (Function Block) (Used to detect stuck communication)
- LAST_RESET_TIME → TIME (Stores the last time a reset occurred)
- CURRENT_TIME → TIME (Holds the current system time)
- LOG_HISTORY → ARRAY OF STRING (Stores previous log messages)
- LOG_INDEX → INT (Tracks the log position)
```
```VB
* =======================================Check if READ_VAR transaction is complete====================== *)
IF (COMMS_MANAGEMENT[0] AND WORD_TO_INT (IN := ACTIVITY_BIT_MASK)) = 0 THEN

    (* Transaction complete; read the two reports from word [1] *)
    CommunicationReport := COMMS_MANAGEMENT[1] AND 16#00FF;   (* LSB *)
    OperationReport     := COMMS_MANAGEMENT[1] / 256;         (* MSB *)

(*========================== Check the reports for errors ============================================== *)
    IF CommunicationReport = 16#00 THEN
        IF OperationReport <> 16#00 THEN
            ERROR_FLAG := TRUE;
            LOG_MESSAGE := CONCAT_STR(
                 'ERROR: READ_VAR operation issue. Operation Report: ',
                 INT_TO_STRING(OperationReport)
            );
        ELSE
            ERROR_FLAG := FALSE;
            RETRY_COUNT := 0;      
            TIMEOUT_COUNTER := 0;  
            LOG_MESSAGE := 'READ_VAR successful';
        END_IF;

    ELSIF CommunicationReport = 16#FF THEN
        ERROR_FLAG := TRUE;
        LOG_MESSAGE := CONCAT_STR(
             'ERROR: READ_VAR failed. Message refused (16#FF). OperationReport: ',
             INT_TO_STRING(OperationReport)
        );

    ELSE
        ERROR_FLAG := TRUE;
        LOG_MESSAGE := CONCAT_STR('ERROR: READ_VAR failed. CommReport: ',
                                  INT_TO_STRING(CommunicationReport));
        LOG_MESSAGE := CONCAT_STR(LOG_MESSAGE, CONCAT_STR(', OperationReport: ',
                                  INT_TO_STRING(OperationReport)));
    END_IF;

(* ============================================ Check array [3] for data recieved === *)
    IF COMMS_MANAGEMENT[3] = 0 THEN
        ERROR_FLAG := TRUE;
        LOG_MESSAGE := 'WARNING: No data received from READ_VAR';

        (* Cancel current READ_VAR transaction and restart it *)
        IF RETRY_COUNT < MAX_RETRY THEN
            COMMS_MANAGEMENT[1] :=  WORD_TO_INT (IN := CANCEL_BIT_MASK);
          
            COMMS_MANAGEMENT[1] := 0;  (* Reset *)
            RETRY_COUNT := RETRY_COUNT + 1;
            TIMEOUT_COUNTER := 0;
        ELSE
            LOG_MESSAGE := 'Max retries reached. READ_VAR disabled.';
            ERROR_FLAG := TRUE;
        END_IF;
    END_IF;

ELSE
    (* === READ_VAR is still in progress === *)
    LOG_MESSAGE := 'READ_VAR in progress...';

    (* Increment timeout counter but stop at limit *)
    IF TIMEOUT_COUNTER < TIMEOUT_LIMIT THEN
        TIMEOUT_COUNTER := TIMEOUT_COUNTER + 1;
    ELSE
        LOG_MESSAGE := 'Timeout reached, canceling READ_VAR transaction.';
        ERROR_FLAG := TRUE;
        COMMS_MANAGEMENT[1] := WORD_TO_INT (IN := CANCEL_BIT_MASK);  (* Force cancel *)
    END_IF;
END_IF;


(*========= Monitor COMMS_MANAGEMENT[0] for changes over time.If it does not change for 5 seconds, toggle a "reset bit ========*)


 IF (COMMS_MANAGEMENT[0] <> Prev_CommsState) OR 
   (COMMS_MANAGEMENT[1] <> Prev_CommReport) OR 
   (COMMS_MANAGEMENT[2] <> Prev_CommStatus) OR 
   (COMMS_MANAGEMENT[3] <> Prev_CommData) THEN
   
    (* Communication is active; reset the stuck detection timer *)
    Prev_CommsState := COMMS_MANAGEMENT[0];
    Prev_CommReport := COMMS_MANAGEMENT[1];
    Prev_CommStatus := COMMS_MANAGEMENT[2];
    Prev_CommData := COMMS_MANAGEMENT[3];

    BitStuckTimer(IN := FALSE);

ELSE
    (* No change detected => start stuck timer *)
    BitStuckTimer(IN := TRUE, PT := T#5S);
    
    IF BitStuckTimer.Q THEN
        LOG_MESSAGE := 'No change in COMMS_MANAGEMENT for 5s => triggering reset.';
        COMMS_MANAGEMENT[0] := COMMS_MANAGEMENT[0] XOR WORD_TO_INT(IN := STUCK_RESET_BIT_MASK);
    END_IF;
END_IF;

IF BitStuckTimer.Q THEN
    LOG_MESSAGE := 'No change in COMMS_MANAGEMENT for 5s => triggering reset.';
    COMMS_MANAGEMENT[0] := COMMS_MANAGEMENT[0] XOR WORD_TO_INT(IN := STUCK_RESET_BIT_MASK);

    (* Reset stuck timer after performing the reset *)
    BitStuckTimer(IN := FALSE);
END_IF;
IF (BitStuckTimer.Q) AND (LAST_RESET_TIME + T#5S < CURRENT_TIME) THEN
    LOG_MESSAGE := 'No change in COMMS_MANAGEMENT for 5s => triggering reset.';
    COMMS_MANAGEMENT[0] := COMMS_MANAGEMENT[0] XOR WORD_TO_INT(IN := STUCK_RESET_BIT_MASK);
    
    (* Update last reset time to prevent excessive resets *)
    LAST_RESET_TIME := CURRENT_TIME;
END_IF;
IF ERROR_FLAG THEN
    LOG_HISTORY[LOG_INDEX] := LOG_MESSAGE;
    LOG_INDEX := (LOG_INDEX + 1) MOD MAX_LOG_ENTRIES;  (* Circular buffer *)
END_IF;
```
