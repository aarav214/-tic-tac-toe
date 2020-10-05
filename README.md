// Uses the Tic's C API to send and receive data from a Tic.
// NOTE: The Tic's control mode must be "Serial / I2C / USB".
 
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <tic.h>
 
bool handle_error(tic_error * error)
{
  if (error == NULL) { return false; }
  fprintf(stderr, "Error: %s\n", tic_error_get_message(error));
  tic_error_free(error);
  return true;
}
 
// Opens a handle to a Tic that can be used for communication.
//
// To open a handle to any Tic:
//   tic_handle * handle = open_handle(NULL);
// To open a handle to the Tic with serial number 01234567:
//   tic_handle * handle = open_handle("01234567");
tic_handle * open_handle(const char * desired_serial_number)
{
  tic_handle * handle = NULL;
 
  // Get a list of Tic devices connected via USB.
  tic_device ** list = NU LL;
  size_t count = 0;
  tic_error * error = tic_list_connected_devices(&list, &count);
  if (handle_error(error)) { goto cleanup; }
 
  // Iterate through the list and select one device.
  tic_device * device = NULL;
  for (size_t i = 0; i < count; i++)
  {
    tic_device * candidate = list[i];
 
    if (desired_serial_number)
    {
      const char * serial_number = tic_device_get_serial_number(candidate);
      if (strcmp(serial_number, desired_serial_number))
      {
        // Found a device with the wrong serial number, so continue on to
        // the next device in the list.
        continue;
      }
    }
 
    // Select this device as the one we want to connect to, and break
    // out of the loop.
    device = candidate;
    break;
  }
 
  if (device == NULL)
  {
    fprintf(stderr, "Error: No device found.\n");
    goto cleanup;
  }
 
  error = tic_handle_open(device, &handle);
  if (handle_error(error)) { goto cleanup; }
 
cleanup:
  for (size_t i = 0; i < count; i++)
  {
    tic_device_free(list[i]);
  }
  tic_list_free(list);
  return handle;
}
 
int main()
{
  int exit_code = 1;
  tic_handle * handle = NULL;
  tic_variables * variables = NULL;
 
  handle = open_handle(NULL);
  if (handle == NULL) { goto cleanup; }
 
  tic_error * error = tic_get_variables(handle, &variables, false);
  if (handle_error(error)) { goto cleanup; }
 
  int32_t position = tic_variables_get_current_position(variables);
  printf("Current position is %d.\n", position);
 
  int32_t new_target = position > 0 ? -200 : 200;
  printf("Setting target position to %d.\n", new_target);
 
  error = tic_exit_safe_start(handle);
  if (handle_error(error)) { goto cleanup; }
 
  error = tic_set_target_position(handle, new_target);
  if (handle_error(error)) { goto cleanup; }
 
  exit_code = 0;  // This program ran successfully.
 
cleanup:
  // Free the resources used by the variables.
  tic_variables_free(variables);
 
  // Call tic_handle_close() to free its resources and because Windows only
  // allows one open handle per device.
  // (Though, in this program, we are about to return from main, so the program
  // will exit and the operating system will do this for us very soon.)
  tic_handle_close(handle);
 
  return exit_code;
}
