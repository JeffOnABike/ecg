Jeff Larson - project proposal writeup

# Objective:
The goal of my project is to efficiently classify episodes of Ventricular Fibrillations (VF) from sample ECG signal recordings originating from 35 test subjects.

The meaning of efficient classification is as follows:
	* With maximum sensitivity (~100%). If there is an episode, it must be classified as positive. Competing technologies advertise a 98% sensitivity benchmark.
	* With optimal precision (>95%). Classifying an interval of ECG as an episode when there is fact no episode occurring is highly undesirable, but not as important as maintaining maximum sensitivity to true episodes.
	* Classification must be done as quickly as possible. The time window of observations used to classify should be minimized as every second is potentially the last the patient has.
	* Classification algorithms need to be computationally quick and simple, minimizing the additional lag of processing time on top of the window of observation.

# Data Engineering:

Data was obtained from Creighton University's Ventricular Tachyarrhythmia Database available on PhysioNet.org. The [PhysioBank ATM](https://physionet.org/cgi-bin/atm/ATM) was useful to query, access, and write data in text and csv formats, which was performed by running a python script with an authorized connection to Amazon's S3. The 35 records of both digitized ECG data and the corresponding annotations are now publicly available from the bucket 'cudbcsvs'. [Here is an example file](https://s3-us-west-1.amazonaws.com/cudbcsvs/cu01_samples_to_csv.csv)

## Notes about the data:

### Samples 

There are 35 sample records of ECG readings. They are easily identified as having the '_samples_to_csv.csv' suffix, and are on the order of MB.
Each contains:
	* ECG in mV
	* 127,232 samples 
	* 250 Hz sampling frequency
	* dt = .004 seconds

***Data Processing***
Problem: many of the sample files have missing ECG data for varying intervals of time. A justified solution needs to be implemented as far as filling the missing data. Possible solutions:
	- Repeat most recent data as an inference to what is happening during the interruption
	- Fill values with a constant or a random distribution of readings. Doing so, however, may have unintended effects features including ECG that will be used to classify episodes.

### Annotations 

There are 35 corresponding records of annotations. The prefix of the files identify the subject record. Each file has '_show_annotations.csv' suffix, and are on the order of KB.
In addition to other features, each annotation file contains the pertinent features to:
	* 'Seconds' - Duration from recording start in seconds. 
		**This corresponds to the ECG sample data**
	* 'Type' observations which are typically N for normal beats and other letters or symbols for aberrations
		e.g. '[' for the onset of a VF episode
	* 'Aux' - descriptive annotations corresponding to aberrations
		e.g. '(VF'	

***Data Processing***
In recasting the data as csvs onto S3, the only modifications were to the annotations file for each, where missing values were filled with "NA"

### N.B. 
* Five records have should be noted as being collected from subjects with pacemakers: 
	cu12, cu15, cu24, cu25, and cu32
* One record (cu01) was obtained using long-term ECG (Holter) recording, while the rest were digitized from high-level analog signals from patient monitors.

As features are engineered moving forward, some of the variations/inconsistencies in the data as well as consequences of filling missing values will be considered.

# Testing Schema:

## Methods
There are 49 distinct episodes of VF according to annotation files.For individual subject records, this varies between 0 for some (patients cu02, cu14), to 1-5 episodes for each of the other 33 subjects.

VT is also recorded and noted, and will be the subject of further work, however it only occurred in one patient (cu02), where it occurred five times.

Testing the classification algorithms will occur by considering every rolling observation (t_start, t_end) window of k seconds, computed once per second until the end of the record (arbitrarily decided for the time being). For example, applying a rolling observation window of k = 10 seconds over a 509 second sample record would make 500 different computations (with each observation window after the initial overlapping k - 1 seconds with the last observation window). 

Binary classification will be recorded: 

y_hat = '1' for an observation showing signature of VF 
y_hat = '0' for an observation of ECG data deemed non-VF, whether it is normal or another arrhythmia

## Scoring:

	* 'True Positive' will be deemed as a '1' label being applied to an observation window (t_start, t_finish) which includes and actual episode of VF, being after the time of VF onset (t_vfon). The additional constraint important to the context of this problem is that the classification occurs within a maximum amount (t_max) seconds of the annotated onset of the episode. This quantity of time must be not only squarely within the window of time in which life-saving defibrillation is most effective, but also be competitive with that of existing technologies.
	CRITERIA:
		(y_hat == 1) & (t_finish > t_vfon) 

	* 'False Negative' won't be categorized as such unless the classification label is '0' and the time duration of the VF episode (t_vf) exceeds t_max. 
	CRITERIA:
		(y_hat == 0) & (t_finish > t_vfon + t_max)

	* 'True Negative' is '0' label outside of a VF episode or up to k seconds within it. The latter allowance inherently is giving the algorithm an allowance of delayed time equal to the size of its observation window until it is penalized for not classifying a positive.
	CRITERIA:
		(y_hat == 0) & (t_finish < t_vfon + k) 

	* 'False Positive' will be only when a '1' label is classifying an observation window that is not actually overlapping an annotated VF episode.
	CRITERIA:
		(y_hat == 0) & (t_finish < t_vfon + k) 

N.B.: no definition is yet made for how to handle the denoument of a VF episode, where the observation window includes both a section of the VF as well as resumed non-VF ECG activity.


## Evaluation:

Before cross-validating with rolling observation window of duration = k seconds, two assumptions must be made:
	* Calculation frequency: 1 Hz
	* t_max: ?? the tipping point of identifying VF given the life or death implications.

Based on the calculation frequency, there will be approximately 17,500 unique classifications across the 35 sample files.

The goal of the classification model will be, based on the above scoring and assumptions, to capture a minimum of 48/49 episodes (sensitivity > 98%), while keeping a precision of classification above 95%.

The calculation of the observation window must be additionally considered in algorithm selection, since the algorithm will ultimately be applied to real-time data.

