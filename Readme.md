# PHPMyAdmin (Web based mysql admin tool)
![alt text](https://upload.wikimedia.org/wikipedia/commons/thumb/4/4f/PhpMyAdmin_logo.svg/1200px-PhpMyAdmin_logo.svg.png)

It is very likely that your MySQL Server is only available from within the Kubernetes cluster, as services like this shouldn’t be exposed to the public Internet. However, you will need a solution to manage your databases, and one that doesn’t include running exec to gain access to the pod or pods directly.

PhpMyAdmin is a very popular web frontend for managing MySQL servers. It is a good candidate for being able to manage the MySQL servers hosted in Kubernetes, as it enables us to manage all of our databases easily and effectively.

# Getting started
In order to get phpmyadmin on kubernetes there are couple to presequites we need to have such as:
  
**Secret**: Our PhpMyAdmin deployment will link to an already existing MySQL service, rather than allowing users to specify the address to the service. PhpMyAdmin will need to know the root password of the MySQL server it is connecting to.
  
*    A secret has already been created to store the root password for MySQL. We will target our PhpMyAdmin at this secret, that is shared with the MySQL pods. An example of the secret data is shown below for reference.

        ```
        ---
        apiVersion: v1
        kind: Secret
        metadata:
        name: mysql-secrets
        type: Opaque
        data:
        root-password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
        ```
* The root-password value is a arbitrary key name created to store a base64 encoded string of the MySQL server’s root password. All secret values in a Secret resource file must be base64 encoded.
* The PhpMyAdmin Kubernetes deployment will reference the root-password key to set the MYSQL_ROOT_PASSWORD environment variable, when the pod’s container starts.

# Create a Deployment Resource
* Create a new named deployement.yaml

    ```
    touch manifests/deployment.yaml
    ```
* Add the following contents to the deployment.yaml file.

    ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: mysql
    labels:
        app: mysql
    spec:
    strategy:
        type: Recreate
    replicas: 1
    selector:
        matchExpressions:
        - {key: tier, operator: In, values: [mysql]}
    template:
        metadata:
        labels:
            app: mysql
            tier: mysql
        spec:
        containers:
        - image: mysql:5.6
            name: mysql
            env:
            - name: MYSQL_DATABASE
            value: $MYSQL_DATABASE
            - name: MYSQL_USER
            value: $MYSQL_USER
            - name: MYSQL_PASSWORD
            value: $MYSQL_PASSWORD
            - name: MYSQL_ROOT_PASSWORD
            valueFrom:
                secretKeyRef:
                name: mysql-secrets
                key: root-password
            ports:
            - containerPort: 3306
            name: mysql
            volumeMounts:
            - name: mysql-persistent-storage
            mountPath: /var/lib/mysql
        volumes:
        - name: mysql-persistent-storage
            persistentVolumeClaim:
            claimName: mysql

    ```

* Create the deployment using kubectl apply command

    ```
    Kubectl apply -f deployment.yaml
    ```

# Create a Service Resource

Kubernetes pods are ephemeral and their IP address lives only as long as the pod does. This is problematic for long-term accessibility to PhpMyAdmin, which is why we create a service resource for our pods.

A service resource will be assigned a static IP address, and all requests to this IP address will be forwarded to the backend PhpMyAdmin pods. The service resources is coupled to the pods via labels.

* create a new file named service.yaml

    ```
    touch manifests/service.yaml
    ```
* Add the following contents to it.Since our PhpMyAdmin pods are assigned a label of phpmyadmin, we will use the selector in the service.
  
    ```
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: phpmyadmin
    spec:
    type: ClusterIP
    selector:
        app: phpmyadmin
    ports:
    - protocol: TCP
        port: 80
        targetPort: 80
    ```
* To create the service resouce we use the kubectl apply command.
  
    ```
    kubectl apply -f service.yaml
    ```

* That's it :)



  **Note:** ```This deployment uses phpmyadmin:5.0.1 and had issues with nginx ingress like path based routing was not working. Hence, please proceed with caution when using any other version of phpmyadmin. Make sure to specify PMA_ABSOLUTE_URI in your deployment manifest or else your routing will not work properly. Please follow this link for detailed description:``` https://stackoverflow.com/questions/63913233/nginx-rewrite-target-overwritting-domain-suffix-index-php-to-domain-index-php/63941408#63941408 



  