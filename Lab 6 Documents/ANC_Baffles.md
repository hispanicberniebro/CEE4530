
```python
from aide_design.play import*
import scipy
from scipy import special
from scipy.optimize import curve_fit
import collections
import os
import tkinter as tk
from tkinter import filedialog
root = tk.Tk()
root.withdraw()

def aeration_data(DO_column):
    """ This function extracts the data from folder containing tab delimited files of aeration data.
    The file must be the original tab delimited file.
    All text strings below the header must be removed from these files.
    The file names must be the air flow rates with units of micromoles/s.
    An example file name would be "300.xls" where 300 is the flowr ate in micromoles/s
    The function opens a file dialog for the user to select the directory containing the data.
    Parameters
    ----------
    DO_column: index of the column that contains the dissolved oxygen concentration data.
    Returns
    -------
    filepaths: list of all file paths in the directory sorted by flow rate
    airflows: sorted numpy array of air flow rates with units of micromole/s attached
    DO_data: sorted list of numpy arrays. Thus each of the numpy data arrays can have different lengths to accommodate short and long experiments
    time_data: sorted list of numpy arrays containing the times with units of seconds. Each
    """

    dirpath = filedialog.askdirectory()

    #return the list of files in the directory
    filenames = os.listdir(dirpath)
    #extract the flowrates from the filenames and apply units
    airflows=((np.array([i.split('.', 1)[0] for i in filenames])).astype(np.float32))
    #sort airflows and filenames so that they are in ascending order of flow rates
    idx   = np.argsort(airflows)
    airflows = (np.array(airflows)[idx])*u.umole/u.s
    filenames = np.array(filenames)[idx]

    filepaths = [os.path.join(dirpath, i) for i in filenames]
    #DO_data is a list of numpy arrays. Thus each of the numpy data arrays can have different lengths to accommodate short and long experiments
    # cycle through all of the files and extract the column of data with oxygen concentrations and the times
    DO_data=[Column_of_data(i,0,-1,DO_column,'mg/L') for i in filepaths]
    time_data=[(ftime(i,0,-1)).to(u.s) for i in filepaths]
    aeration_collection = collections.namedtuple('aeration_results','filepaths airflows DO_data time_data')
    aeration_results = aeration_collection(filepaths, airflows, DO_data, time_data)
    return aeration_results

def O2_sat(Pressure_air,Temperature):
    """
    This equation is valid for 278 K < T < 318 K
    Parameters
    ----------
    Pressure_air: air pressure with appropriate units.
    Temperature: water temperature with appropriate units
    Returns
    -------
    Saturated oxygen concentration in mg/L
    """
    fraction_O2 = 0.21
    Pressure_O2 = Pressure_air *fraction_O2
    return (Pressure_O2.to(u.atm).magnitude)*u.mg/u.L*np.exp(1727/Temperature.to(u.K).magnitude - 2.105)

def Gran(data_file_path):
    """ This function extracts the data from a ProCoDA Gran plot file.
    The file must be the original tab delimited file.
    Parameters
    ----------
    data_file_path : string of the file name or file path.
    If the file is in the working directory, then the file name is sufficient.
    Example data_file_path = 'Reactor_data.txt'
    Returns
    -------
    V_titrant (mL) as numpy array
    ph_data as numpy array (no units)
    V_sample (mL) volume of the original sample that was titrated
    Normality_titrant (mole/L) normality of the acid used to titrate the sample
    V_equivalent (mL) volume of acid required to consume all of the ANC
    ANC (mole/L) Acid Neutralizing Capacity of the sample
    """
    df = pd.read_csv(data_file_path,delimiter='\t',header=5)
    V_t = np.array(pd.to_numeric(df.iloc[0:,0]))*u.mL
    pH = np.array(pd.to_numeric(df.iloc[0:,1]))
    df = pd.read_csv(data_file_path,delimiter='\t',header=-1,nrows=5)
    V_S = pd.to_numeric(df.iloc[0,1])*u.mL
    N_t = pd.to_numeric(df.iloc[1,1])*u.mole/u.L
    V_eq = pd.to_numeric(df.iloc[2,1])*u.mL
    ANC_sample = pd.to_numeric(df.iloc[3,1])*u.mole/u.L
    Gran_collection = collections.namedtuple('Gran_results','V_titrant ph_data V_sample Normality_titrant V_equivalent ANC')
    Gran = Gran_collection(V_titrant=V_t, ph_data=pH,V_sample=V_S, Normality_titrant=N_t, V_equivalent=V_eq, ANC=ANC_sample )
    return Gran;


def ftime(data_file_path,start,end):
    """ This function extracts the column of times from a ProCoDA data file.
    The file must be the original tab delimited file.
    Parameters
    ----------
    data_file_path : string of the file name or file path.
    If the file is in the working directory, then the file name is sufficient.
    Example data_file_path = 'Reactor_data.txt'
    start: index of first row of data to extract from the data file
    end: index of last row of data to extract from the data
    If the goal is to extract the data up to the end of the file use -1
    Returns
    -------
    numpy array of experimental times starting at 0 day with units of days.
    """
    df = pd.read_csv(data_file_path,delimiter='\t')
    start_time = pd.to_numeric(df.iloc[start,0])*u.day
    day_times = pd.to_numeric(df.iloc[start:end,0])
    time_data = np.subtract((np.array(day_times)*u.day),start_time)
    return time_data;

def Column_of_data(data_file_path,start,end,column,units):
    """ This function extracts a column of data from a ProCoDA data file.
    The file must be the original tab delimited file.
    Parameters
    ----------
    data_file_path : string of the file name or file path.
    If the file is in the working directory, then the file name is sufficient.
    Example data_file_path = 'Reactor_data.txt'
    start: index of first row of data to extract from the data file
    end: index of last row of data to extract from the data
    If the goal is to extract the data up to the end of the file use -1
    column: index of the column that you want to extract. Column 0 is time.
    The first data column is column 1.
    units: string of the units you want to apply to the data.
    Example 'mg/L'S
    If an empty string, '', is passed then no units are applied.
    Returns
    -------
    numpy array of experimental data with the units applied.
    """
    df = pd.read_csv(data_file_path,delimiter='\t')
    if units == '':
        data = np.array(pd.to_numeric(df.iloc[start:end,column]))
    else:
        data = np.array(pd.to_numeric(df.iloc[start:end,column]))*u(units)
    return data;

def notes(data_file_path):
    """This function extracts any experimental notes from a ProCoDA data file.
    The file must be the original tab delimited file.
    Parameters
    ----------
    data_file_path : string of the file name or file path.
    If the file is in the working directory, then the file name is sufficient.
    Example data_file_path = 'Reactor_data.txt'
    Returns
    -------
    dataframe showing the rows of the data file that contain text notes
    inserted during the experiment.
    Use this to identify the section of the data file that you want to extract.
    """
    df = pd.read_csv(data_file_path,delimiter='\t')
    text_row = df.iloc[0:-1,0].str.contains('[a-z]','[A-Z]')
    text_row_index = text_row.index[text_row == True].tolist()
    notes = df.loc[text_row_index]
    return notes


#carbonates
#The following code defines the carbonate system and provides functions for calculating Acid Neutralizing Capacity.
Kw = 10**(-14) * (u.mole/u.L)**2
K1_carbonate = 10**(-6.37)*u.mol/u.L
K2_carbonate = 10**(-10.25)*u.mol/u.L
K_Henry_CO2 = 10**(-1.5) * u.mole/(u.L*u.atm)
P_CO2 = 10**(-3.5) * u.atm

def invpH(pH):
    return 10**(-pH)*u.mol/u.L

def alpha0_carbonate(pH):
    alpha0_carbonate = 1/(1+(K1_carbonate/invpH(pH))*(1+(K2_carbonate/invpH(pH))))
    return alpha0_carbonate

def alpha1_carbonate(pH):
    alpha1_carbonate = 1/((invpH(pH)/K1_carbonate) + 1 + (K2_carbonate/invpH(pH)))
    return alpha1_carbonate

def alpha2_carbonate(pH):
    alpha2_carbonate = 1/(1+(invpH(pH)/K2_carbonate)*(1+(invpH(pH)/K1_carbonate)))
    return alpha2_carbonate

def ANC_closed(pH,Total_Carbonates):
    return Total_Carbonates*(alpha1_carbonate(pH)+2*alpha2_carbonate(pH)) + Kw/invpH(pH) - invpH(pH)

def ANC_open(pH):
    return ANC_closed(pH,P_CO2*K_Henry_CO2/alpha0_carbonate(pH))

# Reactors
# The following code is for reactor responses to tracer inputs.
def CMFR(t,C_initial,C_influent):
    """ This function calculates the output concentration of a completely mixed flow reactor given an influent and initial concentration.
    Parameters
    ----------
    C_initial : The concentration in the CMFR at time zero.
    C_influent : The concentration entering the CMFR.
    t: time made dimensionless by dividing by the residence time of the CMFR. t can be a single value or a numpy array.
    Returns
    -------
    Effluent concentration
    """
    return C_influent * (1-np.exp(-t)) + C_initial*np.exp(-t)

def E_CMFR_N(t, N):
    """ This function calculates a dimensionless measure of the output tracer concentration from a spike input to a series of completely mixed flow reactors.
    Parameters
    ----------
    t: time made dimensionless by dividing by the residence time of the reactor. t can be a single value or a numpy array.
    N : The number of completely mixed flow reactors (CMFR) in series. This would logically be constrained to real numbers greater than 1.
    Returns
    -------
    (Concentration * volume of 1 CMFR) / (mass oftracer)
    """
    #make sure that time is dimensionless and not a mixed time unit
    if hasattr(t, 'magnitude'):
      t.ito(u.dimensionless)
    return (N**N)/special.gamma(N) * (t**(N-1))*np.exp(-N*t)

def E_Advective_Dispersion(t, Pe):
    """ This function calculates a dimensionless measure of the output tracer concentration from a spike input to reactor with advection and dispersion.
    Parameters
    ----------
    t: time made dimensionless by dividing by the reactor residence time. t can be a single value or a numpy array.
    Pe : The ratio of advection to dispersion ((mean fluid velocity)/(Dispersion*flow path length))
    Returns
    -------
    (Concentration * volume of reactor) / (mass of tracer)
    """
    #make sure that time is dimensionless and not a mixed time unit
    if hasattr(t, 'magnitude'):
      t.ito(u.dimensionless)
    #replace any times at zero with a number VERY close to zero to avoid divide by zero errors

    t[t==0]=10**(-50)
    return (Pe/(4*np.pi*t))**(0.5)*np.exp((-Pe*((1-t)**2))/(4*t))

def Tracer_CMFR_N(t_seconds, t_bar, C_bar, N):
    """ Used by Solver_CMFR_N. All inputs and outputs are unitless.
    This is The model function, f(x, ...). It takes the independent variable as the first argument and the parameters to fit as separate remaining arguments.
    Parameters
    ----------
    t_seconds : Array of times (units of seconds, but unitless)
    t_bar : Average time spent in the total reactor (units of seconds, but unitless).
    C_bar : (Mass of tracer)/(volume of the total reactor) unitless.
    N : The number of completely mixed flow reactors (CMFR) in series. This would logically be constrained to real numbers greater than 1.
    Returns
    -------
    (C_bar*E_CMFR_N(t_seconds/t_bar, N))
    The model concentration as a function of time
    """

    return C_bar*E_CMFR_N(t_seconds/t_bar, N)

def Solver_CMFR_N(t_data, C_data, theta_guess, C_bar_guess):
    """ Use non-linear least squares to fit the function, Tracer_CMFR_N(t_seconds, t_bar, C_bar, N), to reactor data.
    Parameters
    ----------
    t_data : Array of times with units
    C_data : Array of tracer concentration data with units
    theta_guess : Estimate of time spent in the total reactor with units.
    C_bar_guess : Estimate of (Mass of tracer)/(volume of the total reactor) with units.
    Returns
    -------
    a tuple with theta (units of s), C_bar (same units as C_bar_guess), and N as the best fit to the data.
    """

    C_unitless = C_data.magnitude
    C_units = str(C_bar_guess.units)
    t_seconds = (t_data.to(u.s)).magnitude
    # assume that a guess of 1 reactor in series is close enough to get a solution
    p0 = [theta_guess.to(u.s).magnitude, C_bar_guess.magnitude,1]
    popt, pcov = curve_fit(Tracer_CMFR_N, t_seconds, C_unitless, p0)
    Solver_theta = popt[0]*u.s
    Solver_C_bar = popt[1]*u(C_units)
    Solver_N = popt[2]
    Reactor_results = collections.namedtuple('Reactor_results','theta C_bar N')
    CMFR = Reactor_results(theta=Solver_theta, C_bar = Solver_C_bar, N = Solver_N)
    return CMFR


def Tracer_AD_Pe(t_seconds, t_bar, C_bar, Pe):
    """ Used by Solver_AD_Pe. All inputs and outputs are unitless.
    This is The model function, f(x, ...). It takes the independent variable as the first argument and the parameters to fit as separate remaining arguments.
    Parameters
    ----------
    t_seconds : Array of times (units of seconds, but unitless)
    t_bar : Average time spent in the reactor (units of seconds, but unitless).
    C_bar : (Mass of tracer)/(volume of the reactor) unitless.
    Pe : The Peclet number for the reactor.
    Returns
    -------
    C_bar*E_Advective_Dispersion(t_seconds/t_bar, Pe)
    The model concentration as a function of time
    """

    return C_bar*E_Advective_Dispersion(t_seconds/t_bar, Pe)

def Solver_AD_Pe(t_data, C_data, theta_guess, C_bar_guess):
    """ Use non-linear least squares to fit the function, Tracer_AD_Pe(t_seconds, t_bar, C_bar, Pe), to reactor data.
    Parameters
    ----------
    t_data : Array of times with units
    C_data : Array of tracer concentration data with units
    theta_guess : Estimate of time spent in one CMFR with units.
    C_bar_guess : Estimate of (Mass of tracer)/(volume of one CMFR) with units.
    Returns
    -------
    a tuple with theta (units of s), C_bar (same units as C_bar_guess), and Pe as the best fit to the data.
    """

    C_unitless = C_data.magnitude
    C_units = str(C_bar_guess.units)
    t_seconds = (t_data.to(u.s)).magnitude
    # assume that a guess of 1 reactor in series is close enough to get a solution
    p0 = [theta_guess.to(u.s).magnitude, C_bar_guess.magnitude,5]
    popt, pcov = curve_fit(Tracer_AD_Pe, t_seconds, C_unitless, p0)
    Solver_theta = popt[0]*u.s
    Solver_C_bar = popt[1]*u(C_units)
    Solver_Pe = popt[2]
    Reactor_results = collections.namedtuple('Reactor_results','theta C_bar Pe')
    AD = Reactor_results(theta=Solver_theta, C_bar = Solver_C_bar, Pe = Solver_Pe)
    return AD

```

```python
from aide_design.play import*
from scipy import stats
import Environmental_Processes_Analysis as EPA
import importlib
importlib.reload(EPA)

```
###### Anthony Arce
###### Elle Blake
###### Ajara Cobourne

#### **Laboratory Assignment 5: Reactors**

### Introduction and Objectives
Lakes, water treatment plants, and river segments are often modeled as reactors. As a result, we conducted this experiment to gain a better understanding of contact time in various reactor designs. We modeled a reactor in a small tank and modified it by adding and removing different baffles in series. This allowed us to compare different designs as we attempted to approach the idealized plug flow reactor model. By comparing the contact time of various trials, we were able to determine an optimal number of baffles and baffle shape in order to reach our goal of maximizing the contact time of the contaminant. This experiment can be applied to the real world when working on something like wastewater treatment. In this specific scenario, we want to maximize contact time between the substance that removes the contaminant and the contaminant itself. The longer they are in contact the better the water can be purified. Finally we wished to examine whether the response curve of our experiment matched the theoretical model, which we will analyze with multiple linear regression. We can understand more about this process through the following derivations:
We set up the experiment as a closed system with a small inlet and outlet, added a shot of a conservative tracer and monitored its concentration as it journeyed through the reactor. We sampled at single point to simplify our analysis. To counteract the skew that comes from this method we must look at the relationship between the Peclet number and the variance. If a Peclet number is greater than 10 equation (1.27 and 1.34) can be used
\[Pe = \frac{2 \theta}{\sigma_t ^2}\\

\Delta h =\frac{( \frac{Q_o}{\ n_oK_o} \frac{4}{\pi d_o^2})^2}{\ 2g}\]

\[K_on_o\frac{\pi d_o^2}{\ 4}\sqrt{2g\Delta h} = Q_r\]


Environments with turbulent mixing often have Reynold number values in the hundreds and follow equation (1.31) when they are in series.

$Re_{jet}= \frac{4Q_{reactor}}{n_{port}\pi d_{jet}v }$

Ideally we would try to minimize friction by minimizing headloss

\[Q_o = K_oA_o\sqrt{2g\Delta h}\]




### Procedures

All steps were completed as described in the CEE 4530 Spring 2018 Lab Manual except for the following modifications. For all trials of this experiment the concentration of our red dye tracer was 100 g/L. The pump flow rate was 380 mL/min and the residence time was approximately 7 minutes. Mass concentration and reactor volume were not measured as described in the lab manual, but estimated (see analysis below). NaCl was not added to the reactor either. For trial 1, two equally spaced non-perforated baffles were securely taped (on alternating sides) to the walls of the reactor. The perforated baffles had 4 vertical holes having a 0.75cm diameter. Trial 2 was identical to trial 1, but was performed with four perforated baffles. Trials 3, 4, and 5 were completed with four, six, and eight non-perforated baffles respectively. However, in Trial 5 we were unable to space the baffles evenly for these trials due to space and material constraints (see Figures 1,2,4,5, No images for setup 3).

!(https://github.com/hispanicberniebro/CEE4530/blob/master/Lab%206%20Documents/images/Setup1.jpg)
Figure 1: Experimental setup for Trial 1: 2 non-perforated baffles

!(https://github.com/hispanicberniebro/CEE4530/blob/master/Lab%206%20Documents/images/Setup2.jpg)
Figure 2: Experimental setup for Trial 2: 4 perforated baffles

!(https://github.com/hispanicberniebro/CEE4530/blob/master/Lab%206%20Documents/images/Setup4.jpg)
Figure 4: Experimental setup for Trial 4: 6 non-perforated baffles

!(https://github.com/hispanicberniebro/CEE4530/blob/master/Lab%206%20Documents/images/Setup5.jpg)
Figure 5: Experimental setup for Trial 5: 8 non-perforated baffles

### Data Analysis

```Python
Mass_Dye = ((500 * u.uL) * 100 * u.g / u.L).to(u.mg)

Vol_Reactor = 2.5 * u.L
C_initial = Mass_Dye  / Vol_Reactor
Flow_Rate = (380 * u.mL/u.min).to(u.ml/u.s)
Residence_Time_CMFR = (Vol_Reactor / Flow_Rate).to(u.s)
# Residence_Time_CMFR

trial1_time = ftime('BaffleTest1.xls',0,-1).to(u.s)
trial2_time = ftime('BaffleTest2.xls',0,-1).to(u.s)
trial3_time = ftime('BaffleTest3.xls',0,-1).to(u.s)
trial4_time = ftime('BaffleTest4.xls',0,-1).to(u.s)
trial5_time = ftime('BaffleTest5.xls',0,-1).to(u.s)
print(trial1_time)
trial1_conc = Column_of_data('BaffleTest1.xls',0,-1,1,'mg/L')
trial2_conc = Column_of_data('BaffleTest2.xls',0,-1,1,'mg/L')
trial3_conc = Column_of_data('BaffleTest3.xls',0,-1,1,'mg/L')
trial4_conc = Column_of_data('BaffleTest4.xls',0,-1,1,'mg/L')
trial5_conc = Column_of_data('BaffleTest5.xls',0,-1,1,'mg/L')

Results_1AD = Solver_AD_Pe(trial1_time,trial1_conc,Residence_Time_CMFR, C_initial)

Results_2AD = Solver_AD_Pe(trial2_time,trial2_conc,Residence_Time_CMFR, C_initial)
Results_3AD = Solver_AD_Pe(trial3_time,trial3_conc,Residence_Time_CMFR, C_initial)
Results_4AD = Solver_AD_Pe(trial4_time,trial4_conc,Residence_Time_CMFR, C_initial)
Results_5AD = Solver_AD_Pe(trial5_time,trial5_conc,Residence_Time_CMFR, C_initial)

#Solver_CMFR_N(t_data, C_data, theta_guess, C_bar_guess)
Results_1C = Solver_CMFR_N(trial1_time,trial1_conc,Residence_Time_CMFR, C_initial)
Results_2C = Solver_CMFR_N(trial2_time,trial2_conc,Residence_Time_CMFR, C_initial)
Results_3C = Solver_CMFR_N(trial3_time,trial3_conc,Residence_Time_CMFR, C_initial)
Results_4C = Solver_CMFR_N(trial4_time,trial4_conc,Residence_Time_CMFR, C_initial)
Results_5C = Solver_CMFR_N(trial5_time,trial5_conc,Residence_Time_CMFR, C_initial)

CModel_1 = (Results_1C.C_bar*E_CMFR_N(trial1_time/Results_1C.theta, Results_1C.N)).to(u.mg/u.L)
CModel_2 = (Results_2C.C_bar*E_CMFR_N(trial2_time/Results_2C.theta, Results_2C.N)).to(u.mg/u.L)
CModel_3 = (Results_3C.C_bar*E_CMFR_N(trial3_time/Results_3C.theta, Results_3C.N)).to(u.mg/u.L)
CModel_4 = (Results_4C.C_bar*E_CMFR_N(trial4_time/Results_4C.theta, Results_4C.N)).to(u.mg/u.L)
CModel_5 = (Results_5C.C_bar*E_CMFR_N(trial5_time/Results_5C.theta, Results_5C.N)).to(u.mg/u.L)

ADModel_1 = (Results_1AD.C_bar*E_Advective_Dispersion((trial1_time/Results_1AD.theta).to_base_units(), Results_1AD.Pe)).to(u.mg/u.L)
ADModel_2 = (Results_2AD.C_bar*E_Advective_Dispersion((trial2_time/Results_2AD.theta).to_base_units(), Results_2AD.Pe)).to(u.mg/u.L)
ADModel_3 = (Results_3AD.C_bar*E_Advective_Dispersion((trial3_time/Results_3AD.theta).to_base_units(), Results_3AD.Pe)).to(u.mg/u.L)
ADModel_4 = (Results_4AD.C_bar*E_Advective_Dispersion((trial4_time/Results_4AD.theta).to_base_units(), Results_4AD.Pe)).to(u.mg/u.L)
ADModel_5 = (Results_5AD.C_bar*E_Advective_Dispersion((trial5_time/Results_5AD.theta).to_base_units(), Results_5AD.Pe)).to(u.mg/u.L)
# Plot
#Plot the data and the two model curves.
Trial1 = plt.plot(trial1_time.to(u.min), trial1_conc, 'ro', trial1_time.to(u.min), CModel_1, 'b', trial1_time.to(u.min), ADModel_1, 'g')
plt.xlabel(r'$time (min)$')
plt.ylabel(r'Concentration $\left ( \frac{mg}{L} \right )$')
plt.title('Trial 1: 3 un-perforated baffles')
plt.legend(['Measured','CMFR Model', 'AD Model'])
plt.savefig(r'images\Baf1.jpg')
plt.show()


Trial2 = plt.plot(trial2_time.to(u.min), trial2_conc, 'ro', trial2_time.to(u.min), CModel_2, 'b', trial2_time.to(u.min), ADModel_2, 'g')
plt.xlabel(r'$time (min)$')
plt.ylabel(r'Concentration $\left ( \frac{mg}{L} \right )$')
plt.title('Trial 2: 4 un-perforated baffles')
plt.legend(['Measured','CMFR Model', 'AD Model'])
plt.savefig(r'images\Baf2.jpg')
plt.show()


Trial3 = plt.plot(trial3_time.to(u.min), trial3_conc, 'ro', trial3_time.to(u.min), CModel_3, 'b', trial3_time.to(u.min), ADModel_3, 'g')
plt.xlabel(r'$time (min)$')
plt.ylabel(r'Concentration $\left ( \frac{mg}{L} \right )$')
plt.title('Trial 3: 4 perforated baffles')
plt.legend(['Measured','CMFR Model', 'AD Model'])
plt.savefig(r'images\Baf3.jpg')
plt.show()

Trial4 = plt.plot(trial4_time.to(u.min), trial4_conc, 'ro', trial4_time.to(u.min), CModel_4, 'b', trial4_time.to(u.min), ADModel_4, 'g')
plt.xlabel(r'$time (min)$')
plt.ylabel(r'Concentration $\left ( \frac{mg}{L} \right )$')
plt.title('Trial 4: 6 un-perforated baffles')
plt.legend(['Measured','CMFR Model', 'AD Model'])
plt.savefig(r'images\Baf4.jpg')
plt.show()

Trial5 = plt.plot(trial5_time.to(u.min), trial5_conc, 'ro', trial5_time.to(u.min), CModel_5, 'b', trial5_time.to(u.min), ADModel_5, 'g')
plt.xlabel(r'$time (min)$')
plt.ylabel(r'Concentration $\left ( \frac{mg}{L} \right )$')
plt.title('Trial 5: 8 un-perforated baffles')
plt.legend(['Measured','CMFR Model', 'AD Model'])
plt.savefig(r'images\Baf5.jpg')
plt.show()
##

Pe = np.array([Results_1AD.Pe, Results_2AD.Pe, Results_3AD.Pe, Results_4AD.Pe, Results_5AD.Pe])
N = np.array([Results_1C.N, Results_2C.N, Results_3C.N, Results_4C.N, Results_5C.N])

print(Pe)
print(N)
### Report the values of t* at F = 0.1 for each of your experiments. Do they meet your expectations?

E1 = E_CMFR_N(trial1_time/Results_1C.theta, Results_1C.N)
E2 = E_CMFR_N((trial2_time/Results_2C.theta), Results_2C.N)
E3 = E_CMFR_N(trial3_time/Results_3C.theta, Results_3C.N)
E4 = E_CMFR_N(trial4_time/Results_4C.theta, Results_4C.N)
E5 = E_CMFR_N(trial5_time/Results_5C.theta, Results_5C.N)

F1 = (np.cumsum(E1)/(Results_1C.theta)).magnitude
F2 = (np.cumsum(E2)/(Results_2C.theta)).magnitude
F3 = (np.cumsum(E3)/(Results_3C.theta)).magnitude
F4 = (np.cumsum(E4)/(Results_4C.theta)).magnitude
F5 = (np.cumsum(E5)/(Results_5C.theta)).magnitude

def T_Bar_10Percent(time, F, Theta):
    j = 0
    while (F[j]< 0.1):
      j = j + 1

    return time[j]/Theta

T_Bar1 = T_Bar_10Percent(trial1_time, F1, Results_1C.theta)
T_Bar2 = T_Bar_10Percent(trial2_time, F2, Results_2C.theta)
T_Bar3 = T_Bar_10Percent(trial3_time, F3, Results_3C.theta)
T_Bar4 = T_Bar_10Percent(trial4_time, F4, Results_4C.theta)
T_Bar5 = T_Bar_10Percent(trial5_time, F5, Results_5C.theta)


T_BarsAt10Percent = [T_Bar1, T_Bar2, T_Bar3, T_Bar4, T_Bar5]
print(T_BarsAt10Percent)
```

### Results

Peclet numbers: [10.53600478 10.54333766  5.53475501  9.28524693 14.11370127]

N values: [6.06066189 6.49735332 4.03787393 5.8180678  7.91152394]

Tbar values: [230.0036687999974, 264.0021811199915, 215.00345376000604, 231.0036019199984 209.0037254400084]

T* values: [0.527530642141380, 0.5410033134314013, 0.43860238822551656, 0.5198484669679676, 0.5703401391040229]

Table 1:

|          Parameter                        |   Value   |
| :------------------------------------------ | :------------- |
| Flow Rate (ml/min)                     | 380  |
|    Reactor Volume  (L)                 |2.5   |
|Titrant Concentration (mg/L Red Dye #40)| 100  |   
|Titrant Volume (mL Red Dye #40)         | 0.50 | 
| Residence time                        |  6.579 |


Table 2:
| Trial     | t*   | N    | Pe |
| ---------- | ----- | ------- |-----|
| 1      |  0.53  |6.060  | 10.536
|2      |  0.54  |6.497  | 10.543
|3      |  0.44  | 4.037 |  5.534
|4      |  0.52  | 5.818 |  9.285
|5      |  0.47  |7.9115 | 14.113   


![figure5](images\Baf1.jpg)

Figure 5: Results for Trial 1: 3 non-perforated baffles

![figure6](images\Baf2.jpg)

Figure 6: Results for Trial 2: 2 non-perforated baffles

![figure7](images\Baf3.jpg)

Figure 7: Results for Trial 3: 4 perforated baffles

![figure8](images\Baf4.jpg)

Figure 8: Results for Trial 4: 6 non-perforated baffles

![figure9](images\Baf5.jpg)

Figure 9: Results for Trial 5: 8 non-perforated baffles



### Discussion
Our results will help us to get a full understanding of how modifying the amount of baffles in a reactor affect contact time and how it compares to theoretical models.
Instead of modeling this system as an extremely complex baffled tank, we will examine the mixing in specific baffled reactor sections. We will look at the Peclet number and t* values and try to maximize both of them based on EPA standards. Where

\[t* = t/\theta\]

We can vary several parameters such as the flow rate or diameter to obtain better conditions.
In this experiment we will indirectly measure the efficacy of baffles by comparing the value of t* at F = 0.1.

\[E_t*= \sqrt{\frac{Pe}{t^*4\pi}}exp\frac{-(1-t^*)^2 Pe)}{\ 4t*}\]

\[frac{N^N}{(N-1)!}*t^{N-1}e^{-Nt^*} = E _N_{(t*)}\]        

Where n is the nth reactor in series and F is the integral of equation the E, exit age distribution equation.


The results in trial 1 appeared to most closely follow a plug-flow reactor (Figure 5). The spikes in the graph can likely be explained by free flowing dye that carved their own path and escaped under the baffles. The relatively high peclet number value of 10.5 in addition to the N value of 6.1 suggests that there was less mixing in this trial (Table 2).

Trial 2 was fairly similar to Trial 1 in terms of t*, the time where the cumulative age distribution is 0.1, but ultimately this trial resulted in our highest t* value of .54 (Table 2). This is most likely a result of abnormalities in the data around 5 and a half minutes into the experiment due to tampering with the photometer (Figure 6).


Trial 3 demonstrated a relatively low peclet number and N value in comparison to our other trials (Table 2). As a result, we can infer that there was more mixing throughout this trial, causing our reactor setup to be more closely related to that of a CMFR. Furthermore, there was a significant peak in our measured concentration at around 7 minutes, which was most likely a result of human error in adjusting the sensor's position in the reactor (Figure 7). This trial demonstrated our lowest t* value amongst all of the trials (0.44).

In trial 4, we included 6 baffles, which resulted in peclet and N values of 9.3 and 5.8, respectively (Table 2). In comparison to our other trials, these results suggest that the increase in baffle number caused our reactor to more closely simulate that of a plug flow reactor as opposed to a CMFR (Figure 8).

Finally, in trial 5, we included 8 unperforated baffles in order to reduce the mixing in our reactor as much as possible (Figure 9). This resulted in the highest peclet number and N value of all 5 of the trials (Table 2). These results ultimately confirmed our belief that by increasing the number of baffles and reducing the holes through which the contaminant can flow, we will minimize mixing and ultimately increase the contact time of the "contaminant." Comparing the N and Peclet numbers as we increased the number of baffles helped to decrease dispersion in our model and increase contact time. All the t* values met our expectations and fell within a reasonable range of being "average" according to EPA standards. However there were unexpected spikes in the graphs of Trial 1 and Trial 2 so we suspect the t* for those may not be complete reliable. However, Trial 1's N and t* values are consistent with the values of Trial 4.

Visual observation during the course of the experiment showed that some areas of the reactor did not interact with the dye until unintentional axial mixing occurred. Due to the density of the tracer, some dye simply sank to the bottom of the reactor and either remained there or was transported in a non sinuous manner from inlet to outlet. Our t bar values were not equivalent to our hydraulic residence time which provided there was more evidence of dead volume.

Based on our results, for a full scale chlorine contact tank we would recommend, a significantly higher number of baffles. This would help to decrease dispersion and increase contact time between the chlorine and pathogen. We observed that our Peclet number was larger when we had more baffles so the optimal value within our trials was 8 baffles. However, adding more than 8 baffles, as many as could reasonably and cost-effectively fit inside a full scale reactor would be optimal.


### Conclusions
As highlighted in our results, we found that when comparing our data to a model of a CMFR (Completely Mixed Flow Reactor) in series and to a model of advective dispersion (AD) the trials generally tended towards the AD model. As we modified the experiment by increasing the number of baffles, dispersion decreased as we slowly approached plug flow. This correlates directly with our objective to gain a clearer understanding of contact times across different reactor designs. As a result of our findings in this lab, we are better able to understand one of the most important design considerations for environmental issues such as contaminant removal.


### Suggestion & Comments

The baffles had some trouble sticking to the reactor sides so some of the tracer slipped through the cracks. Additionally, the dye was quite dense and a portion of it just sank to the bottom. The process of emptying and rinsing out the reactor between trials could have been more efficient too. We either had to pour the reactor contents into a large beaker (with a lot of resistance from the tubes connecting it to other parts of the apparatus) or let the reactor drain through the effluent tube into the sink (a slow process). The location where we had to inject the dye was not at the optimal position which gave rise to dead volume in our reactor. It would have been interesting to include an innocuous bacteria and compare the concentration of this bacterial contaminant while varying contact times and methods such as using a highly porous medium or examining the efficacy of batch reactors. An alternate way to analyze this data would be comparing our results to more real-world examples. For instance, we were given the opportunity to create multiple different set ups, some of which included sand and rocks. Having the opportunity to analyze and compare more unique methods to achieve similar results to the baffles would be interesting. Moving the due dates close to the due date (or the morning of) is sometimes better than moving it earlier.
