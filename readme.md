# Tech Spec - Teleport Challenge

## Overview
To be implemented:

1. `Job` library for starting, stopping, and getting status and streamed logs of jobs.
2. API that implements the `Job` library, permitting clients to call the 4 public methods.
3. CLI Client that calls API, using command line args to specify API methods.

## Job Library
`Job` type will include public fields:

- Command
- Args
- ProcessID

##### Implement methods for:

- Start
	- Assure job command exists. Executables can be located in Go using `exec.LookPath()`.
	- Assure that a duplicate command is not currently running by checking "DB".
	- Execute job w/ args.
	- Write job stdout & stderr to specified dir.
	- Set `ProcessID` on `Job`.
- Stop
	- Kill process.
	- Delete job from "DB".
- Get status
	- Check "DB".  
	- Return "running/complete" and logs.
- Stream logs

##### Tradeoffs
- Killing the process, rather than sending SIGINT/SIGQUIT and waiting, comes with the risks associated with kill, e.g. lingering resources. However, it seems like the most user-intuitive option. Future versions could permit users to specify how they want to "stop" a job.
- This implemenatation will prevent duplicate jobs from running simultaneously. There is conceivably some use in permitting redundant jobs, but it requires a more discussion around use cases (e.g. will this tool be used by customers in automation?)

##### Limitations
- For this milestone, mock DB with a global map rather than integrate with live database.

## API
Implement handlers for:

- Start Job (unary)
	- write logs to temp dir
- Stop Job (unary)
- Status (unary)
- Stream Logs (stream)
	- handles ongoing log output for long-running process

Include interceptors for:

- Authorization/Authentication
	- stream & unary interceptors will authenticate user by examining the certificate-bound access token and authorize using role(s) from the access token.

- Authentication
	- a second layer of authentication will be provided via the client-end of mTLS. The API will verify the client cert with the CA.

##### Tradeoffs
- Writing logs to temp files means logs will be lost when temp dirs are purged, but will help prevent filling the hard disk with logs in the event of a runaway process.
- Certificate-Bound Access Tokens are more complex than relying solely on mTLS for authentication, but also more secure.

##### Limitations
- The CA cert will be self-signed for this milestone.

## Client CLI

- Parse command line args into `client <job> <job_args>`.
- Display usage and exit(1) on non-supported command or malformed args.
- Make mocked OAuth request if no token or expired token, storing token in local config file. Stored/returned token will contain claims including role(s) and client cert hash.
- Make API request for start/stop/status/logs, depending on `job` specified.
	- start - display ProcessID
	- stop - display success message
	- status - display process status and current logs
	- logs - stream job output to stdout

- Include client interceptors for:
	- Authorization (stream & unary)

*Examples:*

- start

Command: `client start top -n 2`

Output: `Job: 12345`

- stop

Command: `client stop 12345`

Output: `Job 12345 stopped`

- status

Command: `client status 12345`

Output:

```
Job 12345 running

Processes: 491 total, 3 running, 488 sleeping, 2531 threads                            08:33:06
Load Avg: 2.13, 2.37, 2.51  CPU usage: 4.0% user, 5.89% sys, 90.9% idle
SharedLibs: 389M resident, 68M data, 34M linkedit.
MemRegions: 115316 total, 3751M resident, 134M private, 3922M shared.
PhysMem: 16G used (2259M wired), 251M unused.
VM: 2440G vsize, 2305M framework vsize, 136052(0) swapins, 178252(0) swapouts.
Networks: packets: 2571594/2051M in, 1859599/622M out.
Disks: 1541524/21G read, 1249721/22G written.

PID    COMMAND      %CPU TIME     #TH    #WQ  #PORT MEM    PURG   CMPRS PGRP  PPID  STATE
0      kernel_task  20.5 02:51:26 182/10 0    0     528M+  0B     0B    0     0     running
1823   iTerm2       15.4 11:40.13 8/1    5    367+  116M-  4236K+ 32M-  1823  1     running


Processes: 492 total, 2 running, 490 sleeping, 2524 threads                            08:36:09
Load Avg: 2.27, 2.34, 2.47  CPU usage: 3.90% user, 6.4% sys, 90.4% idle
SharedLibs: 390M resident, 68M data, 34M linkedit.
MemRegions: 115939 total, 3759M resident, 136M private, 3923M shared.
PhysMem: 16G used (2258M wired), 210M unused.
VM: 2444G vsize, 2305M framework vsize, 136052(0) swapins, 178252(0) swapouts.
Networks: packets: 2576368/2052M in, 1863488/623M out.
Disks: 1542339/21G read, 1252689/22G written.

PID    COMMAND      %CPU TIME     #TH   #WQ  #PORT MEM    PURG   CMPRS PGRP  PPID STATE
0      kernel_task  17.2 02:52:04 182/9 0    0     521M   0B     0B    0     0    running
2574   Microsoft Te 9.4  18:40.51 26    1    318   335M   0B     53M   2553  2553 sleeping
```

- logs

Command: `client logs 12345`

Output (streamed):

```
Processes: 491 total, 3 running, 488 sleeping, 2531 threads                            08:33:06
Load Avg: 2.13, 2.37, 2.51  CPU usage: 4.0% user, 5.89% sys, 90.9% idle
SharedLibs: 389M resident, 68M data, 34M linkedit.
MemRegions: 115316 total, 3751M resident, 134M private, 3922M shared.
PhysMem: 16G used (2259M wired), 251M unused.
VM: 2440G vsize, 2305M framework vsize, 136052(0) swapins, 178252(0) swapouts.
Networks: packets: 2571594/2051M in, 1859599/622M out.
Disks: 1541524/21G read, 1249721/22G written.

PID    COMMAND      %CPU TIME     #TH    #WQ  #PORT MEM    PURG   CMPRS PGRP  PPID  STATE
0      kernel_task  20.5 02:51:26 182/10 0    0     528M+  0B     0B    0     0     running
1823   iTerm2       15.4 11:40.13 8/1    5    367+  116M-  4236K+ 32M-  1823  1     running
```

##### Limitations
- Mocked OAuth, rather than integrating with live IdP.

## Security

### Authentication
- mTLS - Use client cert to authenticate with server to verify client.

##### Limitations
- For for purposes of MVP, we will use self-signed cert. Production apps should use a licensed CA.

### Authorization
- Certificate-Bound Access Tokens - Client will authenticate w/ external IdP, receiving a JWT containing user role(s) and client certificate hash in claims. Server will implement interceptor validating JWT and authorizing role for method called.

##### Limitations
-  For purposes of MVP, we will generate and use a mock token in lieu of requiring the client to open a browser and auth with a live IdP.

### Encryption
- Server requests and responses will be encrypted using TLS.
- Server and client certs should be encrypted at rest. A future implementation of a CI/CD pipeline should use Vault to decrypt the certs.

##### Limitations
- For this milestone, we will use unencrypted test certs and not implement CI/CD.

## Deployment
For this milestone, we will not implement CI/CD. The final PR should include a succinct Readme and makefile/starter script.
