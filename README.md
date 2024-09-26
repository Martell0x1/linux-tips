# Linux Booting Sequence
- A bootloader is a software that loads another software (that software is mainly the kernel / application (OTA))
- 1. CPU runs a programe called BIOS (Basic Input output system)(ROM bootloader) , checks for Hardware , ram , disks , short circutes ,checks for your operating system located on which type of disks (display the loadable devices , floppy, usb) , then uploads the first-stage bootloader (MBR)
- 2. Loads the MBR (the 512 bytes)(Master Boot Record) (giving control) , MBR is located on the first-stage of the hard-disk.
    the MBR is consisting of 2 parts:
        1. primary bootloader `initiate some of device driver , loads the second bootloader (GRUP) (giving control)`
        2. partition table  => when installing partions during the installation process of any system (windows / linux)
        3. mbr validation check
- 3. MBR loads the Gurb (grabd united bootloader) software, which lists all oses entries , GRUB loads the kernel in ram (giving control)
- 4. The kernel after that is loaded in the ram:
    1. load device drivers
    2. memory initation
    3. mount file systems (makes entry points to make you able to read it)
    4. run init process
        `
            if(!try_to_run_init_process("/sbin/init") ||
                !try_to_run_init_process("/etc/init") ||
                !try_to_run_init_process("/bin/init") ||
                !try_to_run_init_process("/bin/sh")) 
                    return 0;
            panic ("No wroking init found . Try passing init= option to kernel. "
                    "See Linux Documentaion/admin-guide/init.rst for guidance. ");
        `
    5. when init process is up ... this is the init_main function:
        `
        static
void init_main(void)
{
  CHILD			*ch;
  struct sigaction	sa;
  sigset_t		sgt;
  int			f, st;

  if (!reload) {
  
#if INITDEBUG
	/*
	 * Fork so we can debug the init process.
	 */
	if ((f = fork()) > 0) {
		static const char killmsg[] = "PRNT: init killed.\r\n";
		pid_t rc;

		while((rc = wait(&st)) != f)
			if (rc < 0 && errno == ECHILD)
				break;
		safe_write(1, killmsg, sizeof(killmsg) - 1);
		while(1) pause();
	}
#endif

#ifdef __linux__
	/*
	 *	Tell the kernel to send us SIGINT when CTRL-ALT-DEL
	 *	is pressed, and that we want to handle keyboard signals.
	 */
	init_reboot(BMAGIC_SOFT);
	if ((f = open(VT_MASTER, O_RDWR | O_NOCTTY)) >= 0) {
		(void) ioctl(f, KDSIGACCEPT, SIGWINCH);
		close(f);
	} else
		(void) ioctl(0, KDSIGACCEPT, SIGWINCH);
#endif

	/*
	 *	Ignore all signals.
	 */
	for(f = 1; f <= NSIG; f++)
		SETSIG(sa, f, SIG_IGN, SA_RESTART);
  }

  SETSIG(sa, SIGALRM,  signal_handler, 0);
  SETSIG(sa, SIGHUP,   signal_handler, 0);
  SETSIG(sa, SIGINT,   signal_handler, 0);
  SETSIG(sa, SIGCHLD,  chld_handler, SA_RESTART);
  SETSIG(sa, SIGPWR,   signal_handler, 0);
  SETSIG(sa, SIGWINCH, signal_handler, 0);
  SETSIG(sa, SIGUSR1,  signal_handler, 0);
  SETSIG(sa, SIGSTOP,  stop_handler, SA_RESTART);
  SETSIG(sa, SIGTSTP,  stop_handler, SA_RESTART);
  SETSIG(sa, SIGCONT,  cont_handler, SA_RESTART);
  SETSIG(sa, SIGSEGV,  (void (*)(int))segv_handler, SA_RESTART);

  console_init();

  if (!reload) {
	int fd;

  	/* Close whatever files are open, and reset the console. */
	close(0);
	close(1);
	close(2);
  	console_stty();
  	setsid();

  	/*
	 *	Set default PATH variable.
	 */
  	setenv("PATH", PATH_DEFAULT, 1 /* Overwrite */);

  	/*
	 *	Initialize /var/run/utmp (only works if /var is on
	 *	root and mounted rw)
	 */
	if ((fd = open(UTMP_FILE, O_WRONLY|O_CREAT|O_TRUNC, 0644)) >= 0)
		close(fd);

  	/*
	 *	Say hello to the world
	 */
  	initlog(L_CO, bootmsg, "booting");

  	/*
	 *	See if we have to start an emergency shell.
	 */
	if (emerg_shell) {
		pid_t rc;
		SETSIG(sa, SIGCHLD, SIG_DFL, SA_RESTART);
		if (spawn(&ch_emerg, &f) > 0) {
			while((rc = wait(&st)) != f)
				if (rc < 0 && errno == ECHILD)
					break;
		}
  		SETSIG(sa, SIGCHLD,  chld_handler, SA_RESTART);
  	}

  	/*
	 *	Start normal boot procedure.
	 */
  	runlevel = '#';
  	read_inittab()
    ...
    `
    there's a function `read_inittab()` here it is:
    ....

        `#if DEBUG
  if (newFamily != NULL) {
	INITDBG(L_VB, "PANIC newFamily != NULL");
	exit(1);
  }
  INITDBG(L_VB, "Reading inittab");
#endif

  /*
   *	Open INITTAB and read line by line.
   */
  if ((fp = fopen(INITTAB, "r")) == NULL)
	initlog(L_VB, "No inittab file found");
    ...
    `
    the `INITTAB` macro's value is `/etc/inittab` .. if we open this file we would see the things that init runs after it's up..
    - first thing first the inittab will run this script located in `/etc/init.d/rcS` , but if you boot in emergency mode `-b` it will not run
        this script is responsible for sitting up the PATH variable , the runLevel , mounting /proc
    side notes: The /proc directory in Linux is a virtual filesystem that provides detailed information about the system, including processes, hardware, and kernel settings. It's often referred to as the process information pseudo-filesystem because it doesn't store actual files on disk, but rather, the files are generated dynamically by the kernel.

    Process Information: Each running process in Linux is represented by a directory under /proc named after its process ID (PID). For example, if the process ID is 123, you can find information about it in /proc/123. Inside each process directory, you'll find:

    /proc/<PID>/status: Contains basic information about the process (e.g., memory usage, state).
    /proc/<PID>/cmdline: Shows the command that started the process.
    /proc/<PID>/fd/: A directory containing file descriptors opened by the process.
    /proc/<PID>/stat: Contains detailed statistics about the process, including CPU usage, state, etc.

System-Wide Information:

    /proc/cpuinfo: Provides information about the CPU, including model, cores, speed, and cache size.
    /proc/meminfo: Shows memory usage statistics, including total and available memory.
    /proc/uptime: Shows how long the system has been running.
    /proc/loadavg: Provides the system load averages (1, 5, and 15 minutes).
    /proc/version: Shows the Linux kernel version.

Kernel Parameters:

    /proc/sys/: This directory contains files that control kernel settings. You can read and write to many of these files to change system behavior at runtime. For example:
        /proc/sys/net/: Networking-related kernel parameters.
        /proc/sys/kernel/: Kernel-level settings like hostname, core dump settings, etc.

Virtual Memory Information:

    /proc/self/: A symlink that points to the process currently accessing /proc. This is useful for getting information about the current process.
    
    also it's reposible for sitting up the CTRL-C Trap(google it)
    
so after this /etc/init.d/rcS script runs , now it's time to run the `RunLevels`

- if the system is halt , then it will call the RunLevel 0 , which is a punch of scripts located in `/etc/init.d/rc 0`
- if the system is loged in with single user , then it will call RunLevel 1 , which is a scrips located in `/etc/init.d/rc 1`
- if the system is multi-user (normal) , then it's RunLevel 2-5 , scripts `/etc/init.d/rc 2,3,4,5`
- if the system wil reboot , it's RunLevel 6 , script `/etc/init.d/rc 6`
