<html>
  <body>
    <h1> GitLab scaled runners with AWS EC2. </h1>
    My experience with autoscaling runners with aws ec2.
   
   <h3> Prerequisites </h3>
    
   >   1. A container of gitlab up and running.
   >   2. A container of gitlab runner up and running or gitlab runner installed in your machine.
   
   <h3> Introduction </h3>
  
   >  Mini tutorial on how to implement scaled gitlab runners with EC2 machines,including template configurations 
      and build commands.
    
   <h3> Benefits </h3>
   
   > - Jobs running in parallel
   > - Economy, as it is only used when you need it. (It can be even more economical using spots instances).
   > - You can set the number of machines per time, data or year using cron format.
    
   <h3> Disadvantages </h3>
   
   > - Requested instances take an average of 1 minute to be created.
    
   <h3>  Registering runner </h3>
    
  >    With your container or gitlab-runner installed in your machine, we will register a runner in your gitlab instance.
  >    On your gitlab instance, go to the admin > runners menu and copy URL and registration token.
  >    Now, execute:
    
  ```sh
        If you be in a container of gitlab-runner:
              docker exec gitlab-runner gitlab-runner register \
              --non-interactive \
              --description "Scaled-Runner" \
              --executor "docker+machine" \
              --docker-image docker:20.10.6 \
              --url "<URL-GIT>" \
              --registration-token "TOKEN GIT" \
              --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
              --tag-list "docker" \
              --run-untagged="true" \ 
              --locked="false"
   ```
   ```sh
         If you in gitlab-runner local:
              gitlab-runner gitlab-runner register \
              --non-interactive \
              --description "Scaled-Runner" \
              --executor "docker+machine" \
              --docker-image docker:20.10.6 \
              --url "<URL-GIT>" \
              --registration-token "TOKEN GIT" \
              --docker-volumes /var/run/docker.sock:/var/run/docker.sock \
              --tag-list "docker" \
              --run-untagged="true" \ 
              --locked="false"
   ```
   
  ## Implementation of template configs 
  > When the runners will be registered, we can create a template configurations archives.
  > If you run the register in a container of GitLab-Runner, in the past /etc/gitlab-runner/ a file called config was created.
  
  #### Adding settings
  ```
    1. Add the option clone_url = "url-git" in the [[runnerS]] section.
    It is necessary for us to be able to clone the git repositories on the machines that will be created.
  ```
  ```
    2. Add the options of section [[runners.machine]]:
      2.1 Set the value of the field IdleCount.
          IdleCount is responsible for controlling the number of machines that were idle without 
          requiring a request for it to be active.
      2.2 Set the value of the field IdleTime.
          Idletime is responsible for controlling the time that machines will die after
          they no longer receive requests and the number of machines is greater than specified in IdleCount.
      2.3 Set the value of MaxBuilds.
          MaxBuild is responsible for controlling the max pipeline builds of each machine, if the number of builds
          in the machine is greater than set in MaxBuilds, the machine will die.
      2.4 Set the name of each machine in MachineName. (added a %s at the end of the name to make each name different).
      2.5 set the driver that the machines will be created by the docker-machine in MachineDriver.
          Driver: amazonec2
  ```
  ```
    3. Add machine build options:
      In the section MachineOptions
      Add the following settings through your AWS resources:
    
      "amazonec2-access-key= YOURACESSKEY ",
      "amazonec2-secret-key= YOURSECRETKEY ",
      "amazonec2-region= YOUR-REGION ",
      "amazonec2-zone= YOUR-REGION-ZONE ",
      "amazonec2-vpc-id=vpc-YOUR-VPC-ID",
      "amazonec2-subnet-id=subnet-YOUR-SUBNET-ID",
      "amazonec2-security-group= NAME-OF-THE-SECURITY-GROUP ",
      "amazonec2-instance-type= INSTANCE-TYPE ",
      "amazonec2-volume-type= VOLUME-TYPE ",
      "amazonec2-private-address-only=true",  # FOR USE ONLY PRIVATE ADDRESS ON YOUR MACHINES. (Optional)
      "amazonec2-tags= YOUR TAGS"  # For added tags use "NAME-OF-THE-KEY, VALUE-OF-THE-KEY". 
  ```
  
  > At the end of configurations, your config.toml file will be like this:
   
  ```
concurrent = 10   # Number of competition pipelines for machines, in this case 10 machines run in parallel
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner"
  url = "URL do Gitlab"
  token = "Token do Gitlab"
  executor = "docker+machine"
  clone_url= "URL-GIT"
  limit = 10               # Limit of machines that will be created, in this case only 10 machines will be created
  [runners.docker]
    tls_verify = false
    privileged = true 
    image = "docker:19.03.5"   # Your docker image version
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
    shm_size = 0
  [runners.machine]
    IdleCount = 1     # Machines in the idle state.
    IdleTime = 40     # Time in Seconds.
    MaxBuilds = 3     # Number of maxbuilds.
    MachineName = "gitlab-runner-%s"
    MachineDriver = "amazonec2"
    MachineOptions = [
      "amazonec2-access-key= YOURACESSKEY ",
      "amazonec2-secret-key= YOURSECRETKEY ",
      "amazonec2-region= YOUR-REGION ",
      "amazonec2-zone= YOUR-REGION-ZONE ",
      "amazonec2-vpc-id=vpc-YOUR-VPC-ID",
      "amazonec2-subnet-id=subnet-YOUR-SUBNET-ID",
      "amazonec2-security-group= SECURITY-GROUP-NAME ",
      "amazonec2-instance-type= INSTANCE-TYPE ",
      "amazonec2-volume-type= VOLUME-TYPE ",
      "amazonec2-private-address-only=true",
      "amazonec2-tags=Example,AutoScaling"
    ]

  ```
  > If everything is set up correctly, you should already be able to see your autoscaling runners working,
    running pipelines and verifying machine builds through the docker-machine.
  ``` 
    Run docker-machine ls in terminal or inside a container and observe the machines state creation and return.
  ```
    
  ## Autoscaling by cron format
    
  > In the [[runners.machine]] section, add:
    
  ```sh
    
    # You can set the times of autoscaling machines.
  [[runners.machine.autoscaling]]
    Periods = ["* * 8-11 * * mon-fri *"] # Every day of the week, between 8:00am and 11:59am, there will be an idle machine.
    IdleCount = 1
    IdleTime = 180
    Timezone = "America/Sao_Paulo"  # Your favorite zone
  [[runners.machine.autoscaling]]
    Periods = ["* * 14-17 * * mon-fri *"] # Every day of the week, between 14:00pm and 17:59, there will be an idle machine.
    IdleCount = 1
    IdleTime = 180
    Timezone = "America/Sao_Paulo"
  [[runners.machine.autoscaling]]
    Periods = ["* * * * * sat,sun *"] # On weekends, there will be no idle machines.
    IdleCount = 0
    IdleTime = 60
    Timezone = "America/Sao_Paulo"
    
  ```
  
  </body>
</html>
