# Python3 OpenCV3
Modify from https://github.com/JarveeLee/SynthText_Chinese_version and https://github.com/Snowty/SynthText_Chinese

# Steps
I have tested in Mac Sierra version 10.12.3. Here is my configuration:
  ```
  Python 3.6.4 
  numpy   
  Matlab R2017a    
  Refer to http://blog.sina.com.cn/s/blog_4d8209120102wgue.html to run matlab in the terminal:
  1)vim ~/.profile, add the following 2 lines:
  export PATH=$PATH:/Applications/MATLAB_R2014b.app/bin/
  alias mrun="matlab -nodesktop -nosplash -logfile `date +%Y_%m_%d-%H_%M_%S`.log -r"                        


Firstly:

  Add your own pictures in `/prep_script/my_img/img_dir`
  
  Download DCNF-FCSP: https://bitbucket.org/fayao/dcnf-fcsp/get/f66628a4a991.zip in the `prep_scripts` folder，unzip it and rename with `fayao-dcnf-new`.
   
  Then put `run_ucn.m`、`floodFill.py`、`predict_depth.m` 、`Multiscale Combinatorial Grouping`: https://github.com/jponttuset/mcg/archive/master.zip in `pre_scripts/fayao-dcnf-new/demo`.
  Notice: replace the matconvnet folder under fayao-dcnf-new/libs with matconvnet-1.0-beta14(http://www.vlfeat.org/matconvnet/download/)
Secondly:

  Compute the pictures' depth and segment to make the pictues more realistic.

  ```
  cd prep_scripts/fayao-dcnf-new/demo

  mrun run_ucn                             //   prep_script/my_img/ucm.mat
  python floodFill.py                      //   prep_script/my_img/seg.h5

  cd ../libs/matconvnet/
  matlab -nodesktop
        >> addpath matlab/
        >> mex -setup c++
        >> cd matlab/
        >> vl_compilenn
        >> exit
  cd ../../demo/
  mrun predict_depth	                     //   prep_script/my_img/depth.h5 
  error occured:
  `Error using vl_argparse (line 63)
  OPTS must be a structure
  refer to https://github.com/vlfeat/matconvnet/issues/1070, replace the vl_argparse.m with:
  
  %%%%%%%start of vl_argparse.m%%%%%%%%%%%%%%%%%%%%%%%%%%%
  function [opts, args] = vl_argparse(opts, args, varargin)
%VL_ARGPARSE Parse list of parameter-value pairs.
%   OPTS = VL_ARGPARSE(OPTS, ARGS) updates the structure OPTS based on
%   the specified parameter-value pairs ARGS={PAR1, VAL1, ... PARN,
%   VALN}. If a parameter PAR cannot be matched to any of the fields
%   in OPTS, the function generates an error.
%
%   Parameters that have a struct value in OPTS are processed
%   recursively, updating the individual subfields.  This behaviour
%   can be suppressed by using VL_ARGPARSE(OPTS, ARGS, 'nonrecursive'),
%   in which case the struct value is copied directly (hence deleting any
%   existing subfield existing in OPTS). A direct copy occurrs also if the
%   struct value in OPTS is a structure with no fields. The nonrecursive
%   mode is especially useful when a processing time is a concern.
%
%   One or more of the (PAR, VAL) pairs in the argument list can be
%   replaced by a structure; in this case, the fields of the structure
%   are used as paramater names and the field values as parameter
%   values. This behaviour, while orthogonal to structure-valued parameters,
%   is also disabled in the 'nonrecursive' mode.
%
%   [OPTS, ARGS] = VL_ARGPARSE(OPTS, ARGS) copies any parameter in
%   ARGS that does not match OPTS back to ARGS instead of producing an
%   error. Options specified as structures are passed back as a list
%   of (PAR, VAL) pairs.
%
%   Example::
%     The function can be used to parse a list of arguments
%     passed to a MATLAB functions:
%
%        function myFunction(x,y,z,varargin)
%        opts.parameterName = defaultValue ;
%        opts = vl_argparse(opts, varargin)
%
%     If only a subset of the options should be parsed, for example
%     because the other options are interpreted by a subroutine, then
%     use the form
%
%        [opts, varargin] = vl_argparse(opts, varargin)
%
%     that copies back to VARARGIN any unknown parameter.
%
%   See also: VL_HELP().

% Copyright (C) 2015-16 Andrea Vedaldi and Karel Lenc.
% Copyright (C) 2007-12 Andrea Vedaldi and Brian Fulkerson.
% All rights reserved.
%
% Tishis file is part of the VLFeat library and is made available under
% the terms of the BSD license (see the COPYING file).

if ~isstruct(opts) && ~isobject(opts), error('OPTS must be a structure') ; end
if ~iscell(args), args = {args} ; end

recursive = true ;
if numel(varargin) == 1
  if strcmp(lower(varargin{1}), 'nonrecursive') ;
    recursive = false ;
  else
    error('Unknown option specified.') ;
  end
end
if numel(varargin) > 1
  error('There can be at most one option.') ;
end

optNames = fieldnames(opts)' ;

% convert ARGS into a structure
ai = 1 ;
keep = false(size(args)) ;
while ai <= numel(args)

  % Check whether the argument is a (param,value) pair or a structure.
  if recursive && isstruct(args{ai})
    params = fieldnames(args{ai})' ;
    values = struct2cell(args{ai})' ;
    if nargout == 1
      opts = vl_argparse(opts, vertcat(params,values)) ;
    else
      [opts, rest] = vl_argparse(opts, reshape(vertcat(params,values), 1, [])) ;
      args{ai} = cell2struct(rest(2:2:end), rest(1:2:end), 2) ;
      keep(ai) = true ;
    end
    ai = ai + 1 ;
    continue ;
  end

  if ~isstr(args{ai})
    error('Expected either a param-value pair or a structure.') ;
  end

  param = args{ai} ;
  value = args{ai+1} ;

  p = find(strcmpi(param, optNames)) ;
  if numel(p) ~= 1
    if nargout == 1
      error('Unknown parameter ''%s''', param) ;
    else
      keep([ai,ai+1]) = true ;
      ai = ai + 2 ;
      continue ;
    end
  end
  field = optNames{p} ;

  if ~recursive
    opts.(field) = value ;
  else
    if isstruct(opts.(field)) && numel(fieldnames(opts.(field))) > 0
      % The parameter has a  non-empty struct value in OPTS:
      % process recursively.
      if ~isstruct(value)
        error('Cannot assign a non-struct value to the struct parameter ''%s''.', ...
          field) ;
      end
      if nargout > 1
        [opts.(field), args{ai+1}] = vl_argparse(opts.(field), value) ;
      else
        opts.(field) = vl_argparse(opts.(field), value) ;
      end
    else
      % The parameter does not have a struct value in OPTS: copy as is.
      opts.(field) = value ;
    end
  end

  ai = ai + 2 ;
end

args = args(keep) ;
%%%%%%%%%%end %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%


  cd ../../my_img/
  python add_more_data.py                   //  prep_script/my_img/dset.h5

  cp dset.h5 ../../data/
  ```

Secondly:

  Generate the picture.
  
  ```
  cd ../../ 
  python gen.py --viz
  ```

More details can refer to the following chapters.


# SynthText from Ankush

Modify from https://github.com/ankush-me/SynthText.git to generate chinese character 

My OS is Ubuntu opencv2.4 But I am not sure whether it can run on other OS

I changed some func,just run gen.py will be OK,in gen.py I change the depth prediction map with gray map for generating char on cartoon image , for natural img you need to change back to depth map ,other gen**.py contains similar code with different path I do for myself...

0,Before running this code make sure your OS support unicode for chinese.. which as well cost me hours....Added chinese may not make sense because in English words are saperated by blank meanwhile in chinese words are saperated by meaning. 

1,In synthGen I added a function called is_chinese(char ) to or with is_english to cal num of valid chars.

2,Updated the .tff char style files and the path.txt,then 

3,some utf-8 decoded and encoded for chinese char ....Ah I forgot the details....

4,So you can add more pic into the dataset and check with issue under the anthor to fix mistakes......

5,If you want to add more img , firstly you need to compute the segmentation and depth prediction by the 2 matlab code and 1 python code provided by author, and then use the add_more_data.py to generate a new big dset.h5 , containing all of imgs and their seg and depth, then rerun gen.py to see its performance.

These are some samples I do.

** Synthetic Scene-Text Image Samples**
![Synthetic Scene-Text Samples](Figure_1-1.png "Synthetic Samples")

![Synthetic Scene-Text Samples](Figure_1-2.png "Synthetic Samples")

![Synthetic Scene-Text Samples](Figure_1.png "Synthetic Samples")

Code for generating synthetic text images as described in ["Synthetic Data for Text Localisation in Natural Images", Ankush Gupta, Andrea Vedaldi, Andrew Zisserman, CVPR 2016](http://www.robots.ox.ac.uk/~vgg/data/scenetext/).


** Synthetic Scene-Text Image Samples**
![Synthetic Scene-Text Samples](samples.png "Synthetic Samples")

The library is written in Python. The main dependencies are:

```
pygame, opencv (cv2), PIL (Image), numpy, matplotlib, h5py, scipy
```

### Generating samples

```
python gen.py --viz
```

This will download a data file (~56M) to the `data` directory. This data file includes:

  - **dset.h5**: This is a sample h5 file which contains a set of 5 images along with their depth and segmentation information. Note, this is just given as an example; you are encouraged to add more images (along with their depth and segmentation information) to this database for your own use.
  - **data/fonts**: three sample fonts (add more fonts to this folder and then update `fonts/fontlist.txt` with their paths).
  - **data/newsgroup**: Text-source (from the News Group dataset). This can be subsituted with any text file. Look inside `text_utils.py` to see how the text inside this file is used by the renderer.
  - **data/models/colors_new.cp**: Color-model (foreground/background text color model), learnt from the IIIT-5K word dataset.
  - **data/models**: Other cPickle files (**char\_freq.cp**: frequency of each character in the text dataset; **font\_px2pt.cp**: conversion from pt to px for various fonts: If you add a new font, make sure that the corresponding model is present in this file, if not you can add it by adapting `invert_font_size.py`).

This script will generate random scene-text image samples and store them in an h5 file in `results/SynthText.h5`. If the `--viz` option is specified, the generated output will be visualized as the script is being run; omit the `--viz` option to turn-off the visualizations. If you want to visualize the results stored in  `results/SynthText.h5` later, run:

```
python visualize_results.py
```
### Pre-generated Dataset
A dataset with approximately 800000 synthetic scene-text images generated with this code can be found [here](http://www.robots.ox.ac.uk/~vgg/data/scenetext/).

### [update] Adding New Images
Segmentation and depth-maps are required to use new images as background. Sample scripts for obtaining these are available [here](https://github.com/ankush-me/SynthText/tree/master/prep_scripts).

* `predict_depth.m` MATLAB script to regress a depth mask for a given RGB image; uses the network of [Liu etal.](https://bitbucket.org/fayao/dcnf-fcsp/) However, more recent works (e.g., [this](https://github.com/iro-cp/FCRN-DepthPrediction)) might give better results.
* `run_ucm.m` and `floodFill.py` for getting segmentation masks using [gPb-UCM](https://github.com/jponttuset/mcg).

For an explanation of the fields in `dset.h5` (e.g.: `seg`,`area`,`label`), please check this [comment](https://github.com/ankush-me/SynthText/issues/5#issuecomment-274490044).

### Further Information
Please refer to the paper for more information, or contact me (email address in the paper).

