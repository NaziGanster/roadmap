EGOSC + HEAT Auto Scaling Demo
==============================

.. contents:: Contents:
   :local: 

Pre-conditions
--------------

1. EGO is running well::

    root@devstack1:/tmp# egosh service list
    SERVICE  STATE    ALLOC CONSUMER RGROUP RESOURCE SLOTS SEQ_NO INST_STATE ACTI  
    WEBGUI   STARTED  2     /Manage* Manag* devstac* 1     1      RUN        34    
    ascd     STARTED  6     /Manage* Manag* devstac* 1     1      RUN        36    
    SD       STARTED  8     /Manage* Manag* devstac* 1     1      RUN        37  

2. Prepare the YAML template::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ cat sleep.scale.yml 
    heat_template_version: 2013-05-23
    description: >
      Heat template to deploy EGO services
    resources:
      hadm:
        type: IBMInc::EGO::Service
        properties:
          description: 'data nodes'
          min_instances: 1
          max_instances: 1
          control_policy: {
            start_type: 'AUTOMATIC',
            max_restarts: 10,
            dependency: {
              dtype: 'conditional',
              satisfy: 'STARTED',
              keep: 'STARTED',
              auto_start: 'TRUE',
              svc_name: 'SD',
            }
          }
          allocation_specification: {
            consumer_id: '/ManagementServices/EGOManagementServices',
            resource_specification: {
              resource_group: 'ManagementHosts',
              resource_requirement: 'select(!NTIA64 &amp;&amp; !SOL64)',
            }
          }
          activity_description: {
            htype: 'all',
            activity_specification: {
              command: 'sleep 10000',
              job_controller: 'sleep 10000'
            }
          }
      hadc:
        type: OS::Heat::AutoScalingGroup
        properties:
          min_size: 1
          desired_capacity: 1
          max_size: 3
          resource:
            type: IBMInc::EGO::Activity
            properties:
              svc_name: 'hadc'
              description: 'data nodes'
              control_policy: {
                start_type: 'AUTOMATIC',
                max_restarts: 10,
                dependency: {
                  dtype: 'conditional',
                  satisfy: 'STARTED',
                  keep: 'STARTED',
                  auto_start: 'TRUE',
                  dep_name: 'hadm',
                }
              }
              allocation_specification: {
                consumer_id: '/ManagementServices/EGOManagementServices',
                resource_specification: {
                  resource_group: 'ManagementHosts',
                  resource_requirement: 'select(!NTIA64 &amp;&amp; !SOL64)',
                }
              }
              activity_description: {
                htype: 'all',
                activity_specification: {
                  command: 'sleep 10000',
                  job_controller: 'sleep 10000'
                }
              }
      up-policy:
        type: OS::Heat::ScalingPolicy
        properties:
          auto_scaling_group_id: {get_resource: hadc}
          adjustment_type: change_in_capacity
          scaling_adjustment: 1
      down-policy:
        type: OS::Heat::ScalingPolicy
        properties:
          auto_scaling_group_id: {get_resource: hadc}
          adjustment_type: change_in_capacity
          scaling_adjustment: -1

Create the Stack
----------------

1. Create stack with HEAT::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat stack-create s1 --template-file=./sleep.scale.yml
    +--------------------------------------+------------+--------------------+----------------------+
    | id                                   | stack_name | stack_status       | creation_time        |
    +--------------------------------------+------------+--------------------+----------------------+
    | 12ced74b-5ae0-4adc-8a20-62e2284c4e70 | s1         | CREATE_IN_PROGRESS | 2014-09-29T07:57:43Z |
    +--------------------------------------+------------+--------------------+----------------------+

2. Check stack status, stack was created successfully::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat stack-list
    +--------------------------------------+------------+-----------------+----------------------+
    | id                                   | stack_name | stack_status    | creation_time        |
    +--------------------------------------+------------+-----------------+----------------------+
    | 12ced74b-5ae0-4adc-8a20-62e2284c4e70 | s1         | CREATE_COMPLETE | 2014-09-29T07:57:43Z |
    +--------------------------------------+------------+-----------------+----------------------+
    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat resource-list s1
    +---------------+--------------------------------------+----------------------------+-----------------+----------------------+
    | resource_name | physical_resource_id                 | resource_type              | resource_status | updated_time         |
    +---------------+--------------------------------------+----------------------------+-----------------+----------------------+
    | hadc          | 80dbd10f-73c0-49db-88cc-25184c650e06 | OS::Heat::AutoScalingGroup | CREATE_COMPLETE | 2014-09-29T07:57:43Z |
    | hadm          | hadm                                 | IBMInc::EGO::Service       | CREATE_COMPLETE | 2014-09-29T07:57:44Z |
    | down-policy   | 37af7af9963a48758af8bc59879408e6     | OS::Heat::ScalingPolicy    | CREATE_COMPLETE | 2014-09-29T07:57:46Z |
    | up-policy     | 36d11bc3219d4d9d82bdd13ce94ebcbe     | OS::Heat::ScalingPolicy    | CREATE_COMPLETE | 2014-09-29T07:57:46Z |
    +---------------+--------------------------------------+----------------------------+-----------------+----------------------+

3. EGO Services were created successfully, one hadm (Hadoop Master) and the other is hadc (Hadoop Compute), both of the two services have only one service instance::

    root@devstack1:/tmp# egosh service list
    SERVICE  STATE    ALLOC CONSUMER RGROUP RESOURCE SLOTS SEQ_NO INST_STATE ACTI  
    hadm     STARTED  18    /Manage* Manag* devstac* 1     1      RUN        39
    hadc     STARTED  19    /Manage* Manag* devstac* 1     1      RUN        40
    WEBGUI   STARTED  2     /Manage* Manag* devstac* 1     1      RUN        38    
    ascd     STARTED  6     /Manage* Manag* devstac* 1     1      RUN        36    
    SD       STARTED  8     /Manage* Manag* devstac* 1     1      RUN        37   

Scale Up the Stack
------------------

1. scale up the stack::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat resource-signal s1 up-policy
    jay@devstack1:~/src/jay-work/heat_egosc/service$ 

2. Check scale up result, the hadc (Hadoop Compute) was scale up successfully, you can see there were two service instances for hadc::

    root@devstack1:/tmp# egosh service list
    SERVICE  STATE    ALLOC CONSUMER RGROUP RESOURCE SLOTS SEQ_NO INST_STATE ACTI  
    hadm     STARTED  18    /Manage* Manag* devstac* 1     1      RUN        39    
    hadc     STARTED  19    /Manage* Manag* devstac* 2     1      RUN        40    
                                                           2      RUN        41    
    WEBGUI   STARTED  2     /Manage* Manag* devstac* 1     1      RUN        38    
    ascd     STARTED  6     /Manage* Manag* devstac* 1     1      RUN        36    
    SD       STARTED  8     /Manage* Manag* devstac* 1     1      RUN        37   

3. scale up the stack again::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat resource-signal s1 up-policy
    jay@devstack1:~/src/jay-work/heat_egosc/service$ 

4. Check scale up result, the hadc (Hadoop Compute) was scale up successfully, you can see there were three service instances for hadc::

    root@devstack1:/tmp# egosh service list
    SERVICE  STATE    ALLOC CONSUMER RGROUP RESOURCE SLOTS SEQ_NO INST_STATE ACTI  
    hadm     STARTED  18    /Manage* Manag* devstac* 1     1      RUN        39    
    hadc     STARTED  19    /Manage* Manag* devstac* 3     1      RUN        40    
                                                           2      RUN        41    
                                                           3      RUN        42    
    WEBGUI   STARTED  2     /Manage* Manag* devstac* 1     1      RUN        38    
    ascd     STARTED  6     /Manage* Manag* devstac* 1     1      RUN        36    
    SD       STARTED  8     /Manage* Manag* devstac* 1     1      RUN        37 

Scale Down the Stack
--------------------

1. scale down the stack::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat resource-signal s1 down-policy
    jay@devstack1:~/src/jay-work/heat_egosc/service$ 

2. Check scale down result, the hadc (Hadoop Compute) was scale down successfully, you can see there were two service instances for hadc::::

    root@devstack1:/tmp# egosh service list
    SERVICE  STATE    ALLOC CONSUMER RGROUP RESOURCE SLOTS SEQ_NO INST_STATE ACTI  
    hadm     STARTED  18    /Manage* Manag* devstac* 1     1      RUN        39    
    hadc     STARTED  19    /Manage* Manag* devstac* 2     2      RUN        41    
                                                           3      RUN        42    
    WEBGUI   STARTED  2     /Manage* Manag* devstac* 1     1      RUN        38    
    ascd     STARTED  6     /Manage* Manag* devstac* 1     1      RUN        36    
    SD       STARTED  8     /Manage* Manag* devstac* 1     1      RUN        37  

Delete the Stack
----------------

1. Delete the stack::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat stack-delete s1
    +--------------------------------------+------------+--------------------+----------------------+
    | id                                   | stack_name | stack_status       | creation_time        |
    +--------------------------------------+------------+--------------------+----------------------+
    | 12ced74b-5ae0-4adc-8a20-62e2284c4e70 | s1         | DELETE_IN_PROGRESS | 2014-09-29T07:57:43Z |
    +--------------------------------------+------------+--------------------+----------------------+

2. Check delete result, the stack was deleted successfully::

    jay@devstack1:~/src/jay-work/heat_egosc/service$ heat stack-list
    +----+------------+--------------+---------------+
    | id | stack_name | stack_status | creation_time |
    +----+------------+--------------+---------------+
    +----+------------+--------------+---------------+

3. All of the services related to hadoop cluster was deleted::

    root@devstack1:/tmp# egosh service list
    SERVICE  STATE    ALLOC CONSUMER RGROUP RESOURCE SLOTS SEQ_NO INST_STATE ACTI  
    WEBGUI   STARTED  2     /Manage* Manag* devstac* 1     1      RUN        38    
    ascd     STARTED  6     /Manage* Manag* devstac* 1     1      RUN        36    
    SD       STARTED  8     /Manage* Manag* devstac* 1     1      RUN        37  

