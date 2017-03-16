- Start Date: 2017-03-14
- RFC PR: (leave this empty)
- Origen Issue: (leave this empty)

# Summary

Updates to VectorGenerator to fascilitate abstract implementation of digsrc/cap
on the UltraFlex.  The idea is to use bit.overlay to mark bits for overlay 
(OR digsrc) and provide a method "format_overlay" and an attribute
"overlay_from_mem" to flag whether to implement the action specific to the tester 
object.  Similarly, "store_to_mem" would flag whether tester.store results
in digcap.

# Motivation

The motivation is simply to fascilitate digsrc/cap support in a way that is
flexible enough to be transparent if a tester other than UFlex is being
used. If the tester is a J750, pattern overlay (or HRAM capture) is implemented.  
If the tester is a UFlex, digsrc/cap is implement.  If 93K, the 93K equivalent is
implemented.  Digsrc/cap is heavily used by our organization without this type
of support we would require custom drivers along with other stop gap solutions.

# Detailed design

overlay_from_mem:
Read/Write attribute.  The overlay_from_mem attribute would default to false.  
The UFlex tester object would set it to true.  The app (or pattern) will have 
the option of forcing the attribute to false.  If true (and digsrc supported by
current tester) digsrc is performed instead of inserting global labels.

store_to_mem:
Read/Write attribute.  The equivalent of overlay_from_mem except it enables
digcap.

instrument_setting:
Method for accumulating instrument settings for given pins.  Default behavior
defined in VectorGenerator would be to perform no action.  The Uflex tester
object would be updated to accumulate pins and settings for instrument statement
generation.  This method could be used for more than JUST digsrc/cap.  I could
see our group also using this as the instrument header equivalent of "push_microcode".


format_overlay(dut.pin(:blah), bit.overlay_str, options_hash={}):
Method to create tester specific collateral for overlay (similar to format_pin_state).  Returns a hash with info
for the protocol driver to use in implementing the overlay (or digsrc).  The UFlex
plug-in will accumulate the list of digsrc etc pins and the settings options for that
pin (needs to be unique per instrument pin) for use in generating the instrument header.
The default behavior would be to return info for the J750 style vector modify.
The returned hash would contain information like:
  -required push_microcode (either global label string based on overlay string, or digsrc
  opcode)
  -pin drive state first cycle (.drive_mem or 1 or 0)
  -pin drive state subsequent cycles (.repeat_previous for overlay or .drive_mem for digsrc)
  
  
~~~ruby
def app_code_example
  tester.instrument_setting(dut.pin(:tdi), bit_width: 12)
  dut.reg(:my_reg).bit(:TM).overlay("override_testmode")
  dut.reg(:my_reg).bit(:other_bit).write!(0)
end
~~~

~~~ruby
def protocol_driver_jtag_example
  if bit.has_overlay?
    what_do_i_do_hash = tester.format_overlay(dut.pin(:tdi), bit.overlay_str, serial: true, lsb_first: true, bit_width: 1)
    tester.push_microcode(what_do_i_do_hash[:required_microcode]) if what_do_i_do_hash.key?(:required_microcode)
  
    tester.dont_compress = true
    if what_do_i_do_hash[:state_first_cycle] = :drive_mem
      dut.pin(:tdi).drive_mem
    else
      dut.pin(:tdi).drive(bit.value)
    end
  
    tester.cycle
  
    tester.dont_compress = false
    dut.pin(:tdi).repeat_previous unless what_do_i_do_hash[:state_second_cycle] = :drive_mem
  end
end
~~~

------------------------------------------------------------------------------
Pattern with digsrc (pattern allowed default width of 1 to be used):
------------------------------------------------------------------------------
instruments = {
        (spimosi):DigSrc 1;
}
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


																 

tester.store():
This method would also need an update for UFlex to capture the additional instrument options.
The protocol driver would be updated to do this (JTAG example):
~~~ruby
tester.store!(dut.pin(:tdo), serial: true, lsb_first: true, bit_width: 1)
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



The idea here is that the app has the option (not responsibility) to specify some or all of the 
digsrc/cap settings.  But, the protocol driver has the responsibility (not the option) of telling 
the tester object a little bit about how the protocol works when it calls the provided methods (store and format_overlay).
The tester object has the responsibility of telling the driver how to perform the overlay or digsrc 
(info is provided in the hash returned by format_overlay)  
The tester object accumulates a list of pins used as digsrc/cap and their associated settings.  
Then, the tester object renders the instruments statement at the end based on the settings provided.  
The pattern source *SHOULD* now be able to be rendered for J750 (using vector modify and HRAM) or 
UFlex (using digsrc/cap).


# Drawbacks

Protocol drivers would need an update to use these new attributes/method/method update.
Guides would need to be updated to identify the responsibilities of tester objects and protocol drivers.
Testers plug-in for UFlex will require an update to implement digsrc/cap specifics.


# Unresolved questions

-Need a way to select digsrc or MTO on UFlex?
-Still need a way to get the digsrc start microcode entered 144 cycles before any send microcode
-Need to provide the app or pattern source a way to specify some settings (like
the width of the digsrc or digcap).  Maybe something like:
tester.instrument_setting(dut.pin(:blah), bit_width: 12)
-Can/should format_overlay go ahead and handle the push_microcode, setting pin state and disabling compression?
