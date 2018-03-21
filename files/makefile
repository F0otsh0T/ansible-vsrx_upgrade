# Makefile for creating our standalone Cython program 
INCLUDES=-I/usr/include/python2.7

os_server_actions: os_server_actions.o 
	gcc -o $@ $^ -lpython2.7

os_server_actions.o: os_server_actions.c
	gcc -c os_server_actions.c $(INCLUDES)

 
os_server_actions.c: os_server_actions.py 
	@cython --embed os_server_actions.py  

all: os_server_actions 

clean: 
	@echo Cleaning file 
	@rm -f *~ *.o *.so core core.* *.c os_server_actions

