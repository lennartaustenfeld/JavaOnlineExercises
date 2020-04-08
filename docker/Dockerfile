FROM continuumio/miniconda3:4.8.2


# Install Jupyterhub, Jupyter notebook, OpenJDK, nbgrader, the xeus-cling C++ Kernel and unzip
# https://conda-forge.org/docs/
# https://anaconda.org/conda-forge/nbgrader/labels
# https://nbgrader.readthedocs.io/en/stable/user_guide/installation.html

RUN conda install -c conda-forge jupyterhub notebook openjdk nbgrader xeus-cling unzip


# Install Java kernel / iJava
# https://github.com/SpencerPark/IJava

RUN wget --quiet https://github.com/SpencerPark/IJava/releases/download/v1.3.0/ijava-1.3.0.zip && \
	unzip ijava-1.3.0.zip && \
	python3 install.py --sys-prefix && \
	rm -r java install.py ijava-1.3.0.zip


# Create linux users (default admin and user for nbgrader)
# Set default password nbgpsw

RUN useradd -s /bin/bash --create-home nbgadmin && \
	useradd -s /bin/bash --create-home nbguser && \
	usermod --password \$1\$xg4dbNmA\$ygNrKGsFBsu8hXoGieRSK0 nbgadmin && \
	usermod --password \$1\$xg4dbNmA\$ygNrKGsFBsu8hXoGieRSK0 nbguser && \
	runuser -l nbgadmin -c "/opt/conda/bin/conda init bash" && \
	runuser -l nbguser -c "/opt/conda/bin/conda init bash"


# Disable extensions for non-admins

RUN jupyter nbextension disable --sys-prefix create_assignment/main && \
	jupyter nbextension disable --sys-prefix formgrader/main --section=tree && \
	runuser -l nbgadmin -c '/opt/conda/bin/jupyter nbextension enable --user create_assignment/main' && \
	runuser -l nbgadmin -c '/opt/conda/bin/jupyter nbextension enable --user formgrader/main --section=tree'


# Configure JupyterHub
# https://jupyterhub.readthedocs.io/en/stable/getting-started/security-basics.html
# https://jupyterhub.readthedocs.io/en/stable/getting-started/authenticators-users-basics.html
# https://jupyterhub.readthedocs.io/en/stable/reference/config-user-env.html

RUN mkdir -p /srv/jupyterhub && \
	touch /srv/jupyterhub/jupyterhub_config.py && \
	jupyterhub --generate-config -y True -f /srv/jupyterhub/jupyterhub_config.py && \
	cd /mnt && \
	ln -s /srv/jupyterhub/jupyterhub_config.py


# Configure Jupyter
# https://jupyterhub.readthedocs.io/en/stable/getting-started/config-basics.html
# https://jupyterhub.readthedocs.io/en/stable/reference/config-user-env.html
# https://jupyter-notebook.readthedocs.io/en/stable/config_overview.html

RUN jupyter notebook --generate-config && \
	sed -i 's/^#c.NotebookApp.shutdown_no_activity_timeout.*/c.NotebookApp.shutdown_no_activity_timeout = 60 * 60/' /root/.jupyter/jupyter_notebook_config.py && \
	sed -i 's/^#c.MappingKernelManager.cull_idle_timeout.*/c.MappingKernelManager.cull_idle_timeout = 20 * 60/' /root/.jupyter/jupyter_notebook_config.py && \
	sed -i 's/^#c.MappingKernelManager.cull_interval.*/c.MappingKernelManager.cull_interval = 2 * 60/' /root/.jupyter/jupyter_notebook_config.py \
	&& \
	[ -d /etc/jupyter/ ] || mkdir -p /etc/jupyter/ && \
	mv /root/.jupyter/jupyter_notebook_config.py /etc/jupyter/ && \
	ln -s /etc/jupyter/jupyter_notebook_config.py /mnt/



# Configure nbgrader
# https://nbgrader.readthedocs.io/en/stable/configuration/config_options.html
# https://nbgrader.readthedocs.io/en/stable/user_guide/managing_assignment_files.html#id8

RUN mkdir -p /srv/nbgrader/exchange && \
	chmod ugo+rw /srv/nbgrader/exchange \
	&& \
	echo "c = get_config()\n\
c.CourseDirectory.course_id = \"default-course\"\n\
c.Exchange.root = '/srv/nbgrader/exchange'\n\
c.ExecutePreprocessor.timeout = 300\n\
c.ClearSolutions.code_stub = {\n\
'python': '# YOUR CODE HERE',\n\
'java': '// YOUR CODE HERE',\n\
'C++11': '/* YOUR CODE HERE */',\n\
'C++14': '/* YOUR CODE HERE */',\n\
'C++17': '/* YOUR CODE HERE */',\n\
}" \
> /opt/conda/etc/jupyter/nbgrader_config.py  \
	&& \
	cd /mnt && \
	ln -s /opt/conda/etc/jupyter/nbgrader_config.py



# Create nbgrader accounts for nbgadmin and nbguser
# (Avoids autograding errors, if these accounts submitted solution)

RUN cd /home/nbgadmin/ && \
	runuser -l nbgadmin -c 'export PATH="/opt/conda/bin:$PATH" && nbgrader db student add nbgadmin' && \
	runuser -l nbgadmin -c 'export PATH="/opt/conda/bin:$PATH" && nbgrader db student add nbguser'


# - Default filesystem permissions
#   New homes not readable by others
# - Homes group is nbgadmin for providing feedback
# - Strict permissions for SSL and /mnt
# - Directory to distribute files

RUN sed -i 's/^DIR_MODE=0755.*/DIR_MODE=0770/' /etc/adduser.conf && \
	chmod go-rwx /home/* && \
	chmod g+rwx /home/* \
	&& \
	chgrp nbgadmin /home/* \
	&& \
	chmod o-rwx /mnt \
	&& \
	mkdir /srv/distribution && \
	chown nbgadmin:nbgadmin /srv/distribution/ && \
	echo "hello world!" > /srv/distribution/hello-world.txt && \
	chmod 775 /srv/distribution/hello-world.txt



# Create backup of admin users home

RUN cp -Rp /home/nbgadmin/ /home/.nbgadmin/ && \
	chown nbgadmin:nbgadmin /home/.nbgadmin/

CMD [ "/opt/conda/bin/jupyterhub", "-f", "/srv/jupyterhub/jupyterhub_config.py" ]
