- Start Date: 2017-03-14
- RFC PR: (leave this empty)
- Origen Issue: (leave this empty)

# Summary

Updates to the API to fascilitate abstract implementation of digsrc/cap
on the UltraFlex.  The idea is to use bit.overlay to mark bits for overlay 
(OR digsrc) and provide a method(s) to the app to control how the overlay 
is implemented (if non-default behavior is needed) and method(s) for
protocol drivers to use in implementing the overlay (or digsrc).  Additionally,
a way to configure the capture instrument is needed (on UFlex this is needed for 
rendering the instrument statement).

# Motivation

The motivation is simply to fascilitate digsrc/cap support in a way that is
flexible enough to be transparent if a tester other than UFlex is being
used. If the tester is a J750, pattern overlay (or HRAM capture) is implemented.  
If the tester is a UFlex, digsrc/cap is implement.  If 93K, the 93K equivalent is
implemented.  Digsrc/cap is heavily used by our organization without this type
of support we would require custom drivers along with other stop gap solutions.

# Detailed design

- tester.overlay_style:
Attribute used to control the type of overlay:
~~~ruby
  tester.overlay_style = :subroutine
  tester.overlay_style = :label
  tester.overlay_style = :digsrc
  tester.overlay_style = :mto
~~~



- tester.capture_style:
Attribute used to control the type of capture memory used.
~~~ruby
  tester.capture_style = :digcap
  tester.capture_style = :hram		# or some other descriptive name
  tester.capture_style = :mto
~~~


- tester.overlay:
Method for implementing the tester specific overlay.  The app will have the option
of selecting a non-default overlay type using tester.overlay_style.  The protocol
driver would use this method when overlay is requested:
~~~ruby
  tester.cycle		# Actually generate the drive cycle, whether or not we are going to overlay
  
  if reg_or_val[i].has_overlay?
    options[:pins] = dut.pin(:tdi)	# Tell the tester which pins to overlay
	tester.overlay reg_or_val[i].overlay_str, options
	# ...
	# Same data, but need it on multiple cycles
	tester.cycle
	options[:change_data] = false
	tester.overlay reg_or_val[i].overlay_str, options
  end
~~~
For the UFlex digsrc, this method should insert the digsrc SEND microcode and the
digsrc START microcode (at the beginning of the pattern is fine) if it hasn't already
been inserted.


- tester.source_memory:
Method for configuring the source memory to non-default values.  This will be used
to render the instruments statement on the UFlex.  Need the ability to independently
configure pin/source memory:
~~~ruby
  tester.source_memory :digsrc do |mem|
    mem.pin :tdi, size:32
  end
  
  # If no mem ID given, populate :default configuration:
  tester.source_memory :default do |mem|
    mem.pin :tdi, size: 32
  end
~~~


- tester.capture_memory:
Method for configuring the capture memory to non-default values.  This will be used
to render the instruments statement on the UFlex:
~~~ruby
  tester.capture_memory :digcap do |mem|
    mem.pin :tdo, size: 8, bit_order: :msb0		# LSB first should be default to align with register default
	mem.pin :blah	# ...
  end
  
  # same fall back :default mechanism as tester.source_memory if no mem ID provided
~~~


~~~
------------------------------------------------------------------------------
Pattern with digsrc (pattern allowed default width of 1 to be used):
------------------------------------------------------------------------------
instruments = {
        (spimosi):DigSrc 1;
}
...

((spimosi):DigSrc = Start)                                                                      

...

-- here comes a bit
((spimosi):DigSrc = SEND)                                                                       
                > tp0   000DXXXX ;
repeat 39       > tp0   000DXXXX ;
repeat 40       > tp0   100DLXXX ;
-- here comes the next bit
((spimosi):DigSrc = SEND)                                                                       
                > tp0   000DXXXX ;
repeat 39       > tp0   000DXXXX ;
repeat 40       > tp0   100DLXXX ;
------------------------------------------------------------------------------
end of pattern with digsrc
------------------------------------------------------------------------------
~~~


																 

- tester.store():
This method would also need an update for UFlex to handle digcap or HRAM (or MTO?).

~~~
------------------------------------------------------------------------------
Pattern with digcap (might want 2 digcap instruments with different settings):
  Pattern selected bit width of 32 for spi_miso instead of default of 1
------------------------------------------------------------------------------
instruments = {
        (ADC_CAPTURE_15B):DigCap 15:data_type=long:auto_trig_enable;
        SAR_IN_2:UltraSource;
        (spi_miso):DigCap 32:lsb:serial:data_type=long:auto_trig_enable;
}
...
-- here comes a bit to store (this is a SPI shift, capturing the last cycle of an 80 cycle period)
repeat 40       > tp0   0000XXXX ;
repeat 39       > tp0   1000XXXX ;
((spi_miso):DigCap = Store)                                                                     
                > tp0   1000VXXX ;
-- here comes another bit to store
repeat 40       > tp0   0000XXXX ;
repeat 39       > tp0   1000XXXX ;
((spi_miso):DigCap = Store)                                                                     
                > tp0   1000VXXX ;

------------------------------------------------------------------------------
end of pattern with digcap
------------------------------------------------------------------------------
~~~



The idea here is that the app has the option (not responsibility) to specify some or all of the 
digsrc/cap settings.  The protocol driver doesn't need to know anything about the tester.  The 
tester object does the heavy lifting.  The tester object accumulates a list of pins used as digsrc/cap
and their associated settings.  Then, the tester object renders the instruments statement at the end 
based on the settings provided and what was used.  The pattern source *SHOULD* now be able to be 
rendered for J750 (using vector modify and HRAM) or UFlex (using digsrc/cap).


# Drawbacks

Protocol drivers would need an update to take advantage of the new features.
Guides would need to be updated to explain how to use the new features.
Testers plug-in for UFlex and 93K will require an update to implement digsrc/cap specifics.


# Unresolved questions
