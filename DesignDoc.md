# **Take Home Project**

## **Summary**
----
My interpretation of this exercise is to build a server that stores jobs. These jobs can be run in a worker library that can be remotely managed through a secure channel by those who are authorized. A major assumption I am making is that all jobs have outputs and are only meant to be run once. A stopped or finished job cannot be run again. You would have to schedule a new job in that case. This will simplify the implementation.

## **Worker Library**
----
Firstly, the number of workers will be hardcoded but this could be moved into a configuration file in the future. The worker library will include a variable storing a slice of pointers to `job` instances. The global nature of this state requires not only a lock for the slice variable but one for every `job` instance pointer. Jobs will be stored in memory, as persistent storage will not be implemented. The drawback is that if a server goes down all the in memory data will be wiped. Furthermore, the resulting architecture is meant to handle only small amounts of traffic and not super intensive CPU tasks. This will not provide for a highly available system unless it is vertically or horizontally scaled.

The worker library will export three methods: `Start`, `Stop`, and `Query`.

The `Start` method takes in a `name` and a `filepath` representing the path to the go file that has the job. (I'm assuming that the jobs sit on separate files on the same server.) Furthermore, this method will instantiate a new `job` instance pointer and assign it to a worker through a `jobs` channel.
This `job` instance will have the following signature:

```go
type job struct {
  id int // id to perform actions on jobs
  filePath string // path to file of job and human readability
  status Status // status of the job intialized to "starting" ("starting", "running", "stopped", or "finished")
  ctx context.Context // context for the exec.CommandContext
  cancel context.CancelFunc // cancel func to stop a job
  output interface{} // result of a finished job
  lock sync.Mutex // read/write locks
}
```

The worker will execute the command with the `exec.CommandContext` function to create a stoppable process. At this point the job has a status of `running`. After, a `job` instance is processed by a worker, the worker will lock the `job` set the `status` and `output` to the appropriate values.

The `Stop` method takes in a `job` id, references the `job` pointer and call its cancel function to stop the job. A stopped job will not have an output and its status will be set to `stopped`.

A job that has completed will have an `output` value and with a status of `finished`.

The `Query` method also takes in a job id and will return the relevant job info back to the caller.

Overall, this library was not designed to have workers finish in the order in which they are scheduled. As soon as they finish they will write the results to the `job` instance.

## **API**
----
The HTTP API will run on mTLS and JWT authorization. In particular, TLS 1.3 will be used to set up a secure communication channel. It is not only more secure but also has a shorter handshake than previous versions. The cipher suites to be used are: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA.

A self-signed Root CA will be used as the basis of verification for both the client and server certificates. For convenience, openSSL will be used to generate the keys, CSRs and certificates. In the future, there could be a system in place for the server to generate certificates for the client. Localhost will be used for the common name field of the CSR. In the future, since CSR/certification generation will be handled in the codebase, this will be set to the configured server domain name.

A shared Root CA will be used for convenience, but in the future, we may want separate root CAs and maybe even set up intermediate CAs for additional security. JWTs and the server secret will be pregenerated for authorization. They will not have an expiration time because it is only a proof of concept. In the future, a login endpoint and logout endpoint would be necessary to handle the various states of authorization. JWTs are chosen so that servers do not need to store session data.

The API consists of the following endpoints:

**POST** `/start`
- client sends a json body that has the `name` and the job `filePath` to schedule a new job
- returns the job information

**POST** `/stop`
- client sends a json body of the job id to stop a job
- returns the job information

**GET** `/query`
- returns a list of all scheduled jobs, including their relevant info

## **CLI**
----
The client CLI will store their pregenerated certificate in memory for the mTLS handshake.

The client CLI will store the pregenerated JWT as mentioned and send it in the `authorization` header of every request. The JWT payload will consist of the property `user`, which the server can check. The JWT will be sent through the `authorization` header of each HTTP request using the Bearer schema. However, in the future, the client will be required to send the login info.

