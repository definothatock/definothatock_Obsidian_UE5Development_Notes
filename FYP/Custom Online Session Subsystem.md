
**Online Session Interface in Ureal Engine has Delegates List.**

- Info need to send across the internet, and wait for response. e.g. :
    
    - CreateSession() msg to STEAM, CreateSession() wait.
        
    - STEAM made a session, repones back to CreateSession().
        
    - Interface now iterate through the delegate list and broadcast related
        
    - Callback functions provoked and received session data
![[Online Session Subsystem Example.png]]
