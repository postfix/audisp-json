# Battle Testing
Here are some tests that can be run to see how things perform under weird or heavy conditions


(Stealed form go-audit project https://github.com/slackhq/go-audit/blob/master/BATTLE_TESTING.md)

### Receive buffer

The idea of this test is to see where your system will begin having trouble receiving messages from `kauditd`.
It may be possible to tweak the netlink socket receive buffer and avoid this problem. We have not seen any message
loss as a result of this scenario to date.

- Make a file that a user can't read

    `sudo touch /tmp/nope && sudo chmod 0600 /tmp/nope`
    
- Set your `go-audit.yaml` to the following:

    ```
    rules:
      - -a always,exit -F arch=b32 -S open -S openat -F exit=-EACCES -k access
      - -a always,exit -F arch=b64 -S open -S openat -F exit=-EACCES -k access
    ```
    
- Spawn a bunch of background processes that can't read the file (run the following line many times)

    `while [ true ]; do cat /tmp/nope > /dev/null 2>&1; done &`
    
- Run go audit and observe, it may take a while but you should eventually see the following message

    ```
    Error during message receive: no buffer space available
    ```
- Experiment with the `socket_buffer.receive` value in your `audit` config.
- Spawn a background process that executes a command in a subsecond interval.

    `while [ true ]; do uptime > /dev/null; sleep 0.5; done &`
    
- Observe the output of each process, eventually you should see a log line similar to:

    ```
    Likely missed sequence 100, current 102, worst message delay 0
    ```
