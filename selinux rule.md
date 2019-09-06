







l<em>abel   format</em> :  user:role:type:mls_level

example(location: system/sepolicy/private/initial_sid_contexts):

​	id sysctl_fs <u>u:object_r:unlabeled:s0</u>   : the underline part is a label

--------------------------------------------------------------------------------------------

<em>policy rules format</em>:   allow *domains* *types*:*classes* permissions;

*  *Domain* - A label for the process or set of processes. Also called a domain type as it is just a type for a process.

*  *Type* - A label for the object (e.g. file, socket) or set of objects.

*  *Class* - The kind of object (e.g. file, socket) being accessed.

*  *Permission* - The operation (e.g. read, write) being performed.

  

  

   

  